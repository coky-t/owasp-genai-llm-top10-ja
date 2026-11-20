## LLM01:2026 プロンプトインジェクション (Prompt Injection)

### 説明

**プロンプトインジェクションの脆弱性** は、大規模言語モデル (LLM) への入力 (直接ユーザー入力、取得されたコンテンツ、ツール出力、画像、音声、動画コンテンツ、中間的な推論、永続メモリなど) が、アプリケーション開発者が意図しない方法でモデルの挙動を改変する場合に発生します。LLM は「命令」と「データ」をアーキテクチャ上の区別がない (どちらも同一ストリーム上のトークンである) ため、パラメータ化クエリに相当するような明確なものはありません (NCSC, 2025)。モデルに影響を与える入力は、人間が読める必要はなく、ユーザーから直接届く必要はなく、描画されたインタフェースで視認できる必要はありません。

プロンプトインジェクションの脆弱性は、モデルが入力を処理する方法や、その入力が、システムの他の部分へ不適切にデータや命令を渡すように、モデルを強制できる方法に存在します。デプロイメント時の三つの特性がこれを悪化します。第一、**コンテキストウィンドウのプーリング**: モデルは、システムプロンプト、ユーザー入力、検索されたドキュメント、ツール出力、会話履歴、メモリを単一のトークンストリームとして、信頼境界の強制なしに扱います。第二、**メモリの永続性**: 長期メモリ、RAG コーパス、ベクトルストア、あるいはホスト型メモリサービスに書き込まれるインジェクションは、それらのストアから読み取る後続のすべてのセッションを汚染します。第三、**自律型実行**: モデルの出力がツール呼び出し (ファイルシステム、シェル、電子メール、クラウド API、MCP サーバー、サブエージェント) を駆動する場合、爆風半径はチャット画面からエージェントのツールが到達できるところまでに拡大し、ツール出力はコンテキストウィンドウに再入力し、連鎖的あるいは自己複製的な影響を生じる可能性があります。

プロンプトインジェクションの構造は三つの軸で特徴付けできます。**配信経路 (Delivery surface)** は、それがモデルに到達する方法 (直接入力、取得されたコンテンツ、ツール出力、ツール接続チャネル、永続メモリ) です。**伝播挙動 (Propagation behavior)** は、それが時間や境界を越えて拡散する方法 (単一ショット、多段階キルチェーン、メモリや RAG を通じたクロスセッション、エージェント間の自己複製) です。**エンコーディング (Encoding)** は、悪意のある指示がトークンやピクセルを表現する方法 (プレーンテキスト、Base64 やそのたの難読化、不可視の Unicode、マルチモーダルやステガノグラフィック、低リソース言語) です。どの緩和策を適用するかを選択する前の脅威モデリングの段階で、これらの軸に沿ってシナリオを分解することは有益です。

プロンプトインジェクション成功の深刻度や性質は、モデル運用のビジネス上の文脈や、それが組み込まれたエージェンシーによって異なります。プロンプトインジェクションは以下を含む事態を招く可能性がありますが、それに限定されません。

* 機密情報、システムプロンプトの内容、取得されたプライベート文書、インフラストラクチャの詳細を開示します。
* モデル出力を操作して、偏った内容、有害な内容、あるいは攻撃者が意図した内容を生成し、ダウンストリームシステムやユーザーに影響を及ぼします。
* エージェントが呼び出しを許可されているツールの不正に呼び出して、エージェントがシェル、ファイルシステム、またはクラウド API へのアクセスを持つ場合に任意のコマンド実行や破壊的なアクションにエスカレートします。
* 画像 URL チャネル、描画された出力の隠し Unicode 文字、あるいはツールのログ記録での隠しサイドチャネルを介してデータ流出します。
* メモリや RAG コーパスポイズニングを通じてセッションをまたぐエージェント動作を永続的に侵害します。

*注: プロンプトインジェクションは、推論チャネルの内容を含め、モデルがその出力を通じて漏洩するものを扱う、LLM02:2026 機密情報の開示 (Sensitive Information Disclosure) や、特権アクションに到達するモデル出力の結果を扱う、LLM03:2026 過剰なエージェンシー (Excessive Agency) とは異なります。このエントリは入力境界自体に関心を寄せています。モデル出力がダウンストリームコンポーネントに到達する前のサニタイゼーションとバリデーションは LLM10:2026 不適切な出力処理 (Improper Output Handling) でカバーしています。*

---

### プロンプトインジェクションの種類

#### 直接プロンプトインジェクション

ユーザー、あるいはそのユーザーのアクセス経路を利用する攻撃者は、モデルの挙動を望ましくない方法で変更する入力を与えます。直接プロンプトインジェクションには **意図的なもの** (悪意のあるユーザーが脱獄を仕掛ける) あるいは **意図しないもの** (正当なユーザーがたまたま競合する指示を含むコンテンツを貼り付けたり、LLM の支援に依存するユーザーが無関係なダウンストリーム LLM に対して入力を意図せず最適化する (シナリオ #3)) があります。

脱獄はプロンプトインジェクションの一種であり、攻撃者の目的はモデルをその安全性プロトコルに違反することです。アプリケーションレベルのセーフガードはそれを含むのに役立ちますが、効果的な防止にはモデルのトレーニングと安全性メカニズムを継続的に更新する必要があります。

#### 間接プロンプトインジェクション

モデルは、プロンプトインジェクションとして機能するデータを含む外部ソース (ウェブページ、ドキュメント、電子メール、ツールレスポンス、取得した RAG パッセージ、画像、MCP サーバーの出力、データベース行、課題のタイトル) からコンテンツを取り込みます。ユーザーはそれらの指示を供給しておらず、目にもしていません。配信サーフェスの信頼性プロファイルはどのような防御が現実的かを決定します。

* **信頼できないサーフェス。** パブリックウェブページ、未知の送信者からの電子メール、検索結果。防御側はこれらのソースからのあらゆるものを疑わしいものとして扱う必要があります。ほとんどのプロンプトインジェクション研究はここに焦点を当ててきました。
* **部分的に信頼できるサーフェス。** パブリックバグトラッカー上の課題タイトル、パッケージ README と変更履歴、サードパーティ API レスポンス。ユーザーが取得することを選択しましたが、ユーザーが作成したわけではないコンテンツです。ユーザーはプラットフォームを信頼していますが、必ずしも個々の貢献者を信頼してはいません。
* **信頼できるサーフェス。** 開発者自身のリポジトリ、データベース、内部ドキュメント、メール。開発者は、攻撃者が、公開バグレポートフォームなどの無関係なアップストリームの経路を介して、ここにコンテンツを配置したことに気付かない可能性があります。

共通する構造: 攻撃者がバックエンドを直接侵害する必要がありません。攻撃者は開発者の LLM が読み取る場所にテキストを配置します。そして、開発者の権限で動作するその LLM がその処理を実行してしまいます。チャットのサーフェスのみに焦点を当てる防御では、もっぱらこれを見落としてしまいます。

間接プロンプトインジェクションは、ユーザー自身の LLM インスタンスを、ユーザー自身のバックエンドに対する武器に次第に変えてしまいます。パターン: 攻撃者は、低権限チャネル (公開フォーム、カスタマーチケット、コミュニティへのプルリクエスト) を通じて、ユーザーが信頼する場所にテキストを送信します。そして、ユーザーの昇格したクレデンシャルで動作している際に、ユーザーの MCP 接続型のエージェントや開発者支援がそのテキストを読み込むのを待ちます。攻撃者ではなく、エージェントが権限を要するアクションを実行します (よくある例 #3 を参照してください。また、シナリオ #9 では本番環境の概念実証について説明しています)。

### リスクのよくある例

1. **直接的なプロンプト入力による上書き**: ユーザーメッセージはシステムプロンプトのロールや機能の制限を上書きし、モデルの意図されたスコープ外への開示、生成、動作を行わせます。意図的な入力と意図しない入力の双方を含みます。

2. **取得したコンテンツを通じた間接的なインジェクション**: 攻撃者の指示が RAG パッセージ、ウェブページ、ドキュメント、電子メールに紛れ込み、そのコンテンツがコンテキストウィンドウに到達した際に実行します (例: EmailGPT; INCIBE-CERT, 2024)。

3. **信頼できるサーフェスによる間接的なインジェクション**: 低権限だが信頼できるチャネル (課題トラッカー、フィードバックフォーム、サポートチケット) に仕掛けられたテキストは、ユーザーの LLM をそのユーザー自身の高い権限のクレデンシャルの下で動作させ、リポジトリの窃取、データベースのダンプ、IDE 設定の変更といった、攻撃者が直接実行できないアクションを可能になります (Invariant Labs, 2025; General Analysis, 2025; Rehberger, 2025a)。

4. **マルチモーダルおよびステガノグラフィックのインジェクション**: 画像、音声、動画内の知覚不可能な摂動がエンコーダによって抽出されます (Clusmann ほか, 2025; シナリオ #6 を参照)。

5. **不可視文字インジェクションと抽出**: タグブロック、バリエーションセレクタ、ゼロ幅 Unicode は、無害に見えるテキスト内に命令を持ち込んだり、バイトを抽出できます。2024 年 8 月の M365 Copilot での ASCII スマグリングの概念実証では Slack の MFA コードを抽出しました (Rehberger, 2024)。

6. **クロスセッションメモリおよび RAG コーパスポイズニング**: 永続メモリや RAG コーパス内のひとつの汚染されたエントリは、それを読み取る将来のすべてのセッションに到達します (W. Zou ほか, 2025; シナリオ #4 を参照)。

7. **勾配オラクルとしてのファインチューニングインタフェース ("fun-tuning")**: 攻撃者は、ベンダーのファインチューニング API からサンプルごとの損失を読み取り、ペイロードを最適化します (Gemini での攻撃成功は 65% から 82%)。クローズドウェイトモデルに対してホワイトボックススタイルの最適化をもたらします (Labunets ほか, 2025)。

8. **多言語、エンコード済み、または低リソース言語のペイロード**: 低リソースやコードミックスの入力は攻撃成功を高め、そのスキームで訓練していない分析器をすり抜けます。Base64、ROT13、あるいは絵文字でのエンコーディングは、そのエンコーディングを認識できないフィルタをバイパスします (Hackett ほか, 2025)。

---

### 予防と緩和の方策

Prompt injection is intrinsic to current generative AI: LLMs make no architectural distinction between instructions and data, and their behavior is stochastic, so no reliable prevention mechanism exists today, a position consistent with NIST (2025), NCSC (2025), and Debenedetti et al. (2025). Defense is therefore architectural rather than interceptive. Design the surrounding system on the explicit assumption that the model's instruction boundary will eventually be bypassed, and constrain what the model is permitted to do, and what its outputs are permitted to reach, so a successful injection does not translate into a successful exploit.

Most high-impact prompt-injection incidents on record became severe because the injection landed inside a system whose tools, scopes, or output-rendering capabilities let the compromised model act on the attacker's behalf at the user's privilege level (see Scenarios #7 through #9). This is the operational relationship between this entry and **LLM03:2026 Excessive Agency**: prompt injection is the input-side compromise, and excessive functionality, permissions, or autonomy are what give that compromise consequences outside the chat window. Simon Willison's "lethal trifecta" (2025) restates the same structural diagnosis as a pre-deployment check: an agent that can simultaneously access private data, ingest untrusted content, and communicate externally has the conditions for high-impact exploitation, and removing any one leg removes them.

Apply the controls below as defense-in-depth, since no single control is sufficient. Some reduce injection success and are expected to degrade against adaptive attackers. Others bound the blast radius once injection succeeds, and these are what survive against attackers who can probe the system. For agentic deployments, the least-privilege and capability-budgeting controls (#4, #8) are load-bearing, and **LLM03:2026** carries the full agency-side treatment.

1. **Constrain the model's role and capabilities in the system prompt.** Use declarative allow and deny statements ("assist with X only, do not access Y, do not forward output to external addresses"), not open-ended grants. This is a partial control only: an attacker who infers the prompt can bypass it (Nasr et al., 2025), so pair it with the privilege controls in #4.

2. **Define a strict output schema and validate every response in trusted application code** before any downstream system acts on it, using structural validation rather than a second LLM call. This catches format violations, not semantic manipulation: a schema-valid response can still carry a malicious SQL query or an exfiltration-formatted email body.

3. **Filter at every modality boundary (text, image, audio, structured data), not just text.** Run modality-specific classifiers, OCR over images, and transcription over audio, then apply text filters to the extracted content. Semantic filters are evadable by rephrasing or encoding, and low-resource and code-mixed inputs degrade their accuracy (Hackett et al., 2025).

4. **Hold credentials and state-change capability in application code, not the model, and grant least privilege per operation.** Route privileged calls through a deterministic policy engine that re-validates intent and arguments at execution time. NIST AI 100-2 E2025 and the CISA and Five Eyes OT joint guidance (CISA et al., 2025) frame this deterministic mediation as a baseline procurement expectation. Broad "convenience" permissions and multi-agent hops re-introduce the risk downstream.

5. **Strip tag-block (U+E0000 to E007F), variation-selector (U+FE00 to FE0F), and zero-width (U+200B, U+200C, U+200D, U+2060) characters at every ingest and render boundary.** These are invisible in normal rendering and smuggle instructions or exfiltration bytes (see Common Example #5), and variation-selector variants (Rehberger, 2025c) smuggle arbitrary bytes invisibly. Stripping does not stop visible-text payloads or future steganographic classes.

6. **Pass external content through a structurally separate, provenance-labeled channel** so the model can distinguish data from instructions (S. Chen et al., 2025; Microsoft Research, 2025). This reduces attack success in non-adaptive tests only: an attacker who knows the marking scheme can mimic it, and StruQ was bypassed under adaptive attack (Nasr et al., 2025).

7. **Require explicit human confirmation before any privileged, irreversible, or externally visible action**, surfacing the exact rendered action rather than a summary to the reviewer. Invisible-character smuggling can make the displayed action differ from the executed one (#5), and approval fatigue degrades reviewer judgment at volume.

8. **Budget agent capabilities with the Rule of Two as a floor** (Meta AI, 2025). Treat simultaneous access to (A) untrusted input, (B) sensitive data, and (C) state change or external communication as high-risk: any [A,B,C] agent needs per-action human approval, and [A,B] or [A,C] configurations need an explicit residual-risk assessment (see Scenario #8). NIST AI 100-2 E2025 and the CISA, FBI, NSA, and ACSC OT guidance (CISA et al., 2025) endorse the rule, which is silent on autonomy depth (Noma Security, 2025).

9. **Treat agent memory writes as privileged operations.** Log the causing prompt, classify writes for instruction or role-modification content, and require approval before instruction-bearing memories persist across sessions. A February 2025 Gemini PoC (Rehberger, 2025b) poisoned memory via delayed tool invocation (MITRE, n.d.). Factual entries shade into instructions, and incremental writes can evade per-write classification.

10. **Pin, sign, and verify every MCP server and third-party tool package, audit tool descriptions for hidden instructions, and monitor tool composition.** Treat these as a software supply-chain surface, covered by **LLM04:2026 Supply Chain** for third-party tool packages and by ASI04 Agentic Supply Chain Vulnerabilities for MCP servers and tool registries (see Scenario #9). Pinning does not stop a payload shipped in the pinned version or tool-description poisoning that leaves the version unchanged.

11. **Test against adaptive attackers who have read the deployed defense, and reject static-only attack-success claims.** Baseline with AgentDojo (Debenedetti et al., 2024) and JailbreakBench (Chao et al., 2024), then red-team with the full defense specification disclosed to the testers. Nasr et al. (2025) found static attack success near zero while adaptive attack success exceeded 90% for most of 12 recent defenses (see also LLMail-Inject, Microsoft Security Response Center, 2025).

---

### 攻撃シナリオの例

#### シナリオ #1: 直接インジェクション

An attacker prompts a customer-support chatbot to ignore its guidelines, query private data stores, and send emails, leading to unauthorized access and privilege escalation.
**Anatomy:** (a) direct user input, (b) single-shot, (c) plain text

#### シナリオ #2: 取得したウェブコンテンツを介した間接インジェクション

A user asks an assistant to summarize a web page containing hidden instructions. The model inserts a markdown image whose URL exfiltrates the private conversation to an attacker-controlled domain. The user sees only the rendered image, never the instruction.
**Anatomy:** (a) retrieved web content (indirect), (b) single-shot with image-URL exfiltration, (c) plain text hidden in page source

#### シナリオ #3: 意図しないインジェクション

A job-description PDF embeds an AI-detection instruction. An applicant unknowingly uses an LLM to optimize their resume against it, the model surfaces the instruction, and the recruiting system flags the candidate, a prompt injection with neither party acting maliciously.
**Anatomy:** (a) indirect (document / PDF), (b) single-shot, (c) plain text

#### シナリオ #4: RAG リポジトリポイズニング

An attacker contributes poisoned documents to a corpus the application retrieves over. A matching query returns the modified content, and its instructions alter the output. As few as five poisoned documents have reached roughly 90% attack success against a knowledge base of millions of texts (W. Zou et al., 2025).
**Anatomy:** (a) retrieved content (RAG corpus), (b) cross-session / cross-user, (c) plain text

#### シナリオ #5: ペイロード分割

An attacker splits malicious instructions across multiple resume fields (header, body, attachment) so no single field looks malicious to a per-field classifier. The LLM recombines them at evaluation, and its recommendation is manipulated.
**Anatomy:** (a) direct user input split across fields, (b) single-shot recombined at evaluation, (c) plain text fragmented

#### シナリオ #6: マルチモーダルのステガノグラフィックインジェクション

An attacker embeds an instruction in an image below the human visual threshold. A multimodal model's vision encoder extracts the payload, behavior changes, and the model produces harmful output or an unauthorized tool invocation. This was demonstrated against four frontier vision-language models in oncology imaging (Clusmann et al., 2025) and against general-purpose models via combined visual perturbation and text steering (R. Chen et al., 2025).
**Anatomy:** (a) image input (indirect / multimodal), (b) single-shot, (c) steganographic / pixel-level encoding

#### シナリオ #7: ゼロクリック文書媒介型エージェント窃取

A crafted email triggers an LLM-powered productivity assistant to exfiltrate organizational data with no user interaction. Aim Security demonstrated this against Microsoft 365 Copilot (Reddy & Gujral, 2025), bypassing both the deployed prompt-injection classifier and the link-redaction filter.
**Anatomy:** (a) email / document (indirect), (b) single-shot with tool invocation, (c) plain text with invisible-Unicode exfiltration channel

#### シナリオ #8: エージェントによる破壊的コマンド実行

Two July 2025 events show the same class of impact via different vectors. An attacker committed a destructive system prompt to the Amazon Q VS Code extension repository before AWS reverted it, though the committed code failed to execute due to a syntax error (Amazon Web Services, 2025a). Separately, a runtime injection caused Amazon Q to execute arbitrary code (Amazon Web Services, 2025b). An agent with shell, file-system, or cloud-API access amplifies an injection into a host-impacting incident.
**Anatomy:** (a) supply-chain / compromised system prompt, or runtime indirect injection, (b) persistent cross-session, or single-shot with shell tool execution, (c) plain text

#### シナリオ #9: Trusted-Backend Indirect Injection through MCP

An attacker plants text in a low-privilege channel (a public GitHub issue, a support ticket, or a malicious npm package), and the developer's LLM reads it under elevated credentials. Invariant Labs (2025) exfiltrated private repositories via a poisoned GitHub issue, General Analysis (2025) dumped a production database through Cursor's Supabase MCP server running `service_role` and bypassing row-level security, and the malicious `postmark-mcp` package (Koi Security, 2025; Toulas, 2025) BCC'd email to an attacker across an estimated 300 organizations.
**Anatomy:** (a) trusted-surface indirect (MCP channel: issue, ticket, npm package), (b) multi-step tool-chain, (c) plain text

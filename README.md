# sard_discord_bot

Discord.Net で作ったサーバー向けの Discord Bot です。.NET 10 の Worker Service として動きます。

## コマンド

| コマンド | 内容 |
|---|---|
| `/time` | 現在時刻を表示します。Discord のタイムスタンプ記法で返すので、各ユーザーのタイムゾーンで表示されます |
| `/init` | 入学年度(期)・所属班・ニックネームを、年度選択 → 班選択 → ニックネーム入力の順に設定します。期ロール(例: 2024年度 → `14期`)と班ロール(`開発` / `ハイブリッド` / `衛生` / `外部`)を付け、ニックネームを変更します。ロールがなければ自動で作成します |

## 必要なもの

- [.NET 10 SDK](https://dotnet.microsoft.com/download)
- [Discord Developer Portal](https://discord.com/developers/applications) で作成したアプリケーションと Bot トークン

## セットアップ

### 1. Bot をサーバーに招待する

Developer Portal の OAuth2 → URL Generator で、次を選んで招待 URL を作ります。

- **Scopes**: `bot`、`applications.commands`
- **Bot Permissions**: `Manage Roles`、`Manage Nicknames`

URL を直接組み立てる場合は次のとおりです(`<CLIENT_ID>` はアプリケーション ID)。

```
https://discord.com/oauth2/authorize?client_id=<CLIENT_ID>&scope=bot+applications.commands&permissions=402653184
```

招待後、サーバー設定で **Bot のロールを期・班のロールより上** に置いてください。Bot は自分のロールより下のロール・ユーザーしか操作できません。権限の詳細は [RequiredPermission.md](RequiredPermission.md) にまとめています。

特権 Intent(Presence / Server Members / Message Content)は使っていないので、Developer Portal で有効にする必要はありません。

### 2. トークンを設定する

開発中は user-secrets を使います。

```sh
dotnet user-secrets set "Discord:Token" "<token>"
dotnet user-secrets set "Discord:GuildId" "<id>"   # 任意
```

| キー | 必須 | 説明 |
|---|---|---|
| `Discord:Token` | ○ | Bot トークン。未設定だと起動時にエラーで停止します |
| `Discord:GuildId` | | 指定するとそのサーバーにだけコマンドを登録し、すぐに反映されます。未指定ならグローバル登録になり、反映に時間がかかることがあります |

本番環境では環境変数 `Discord__Token` / `Discord__GuildId` で渡してください。

> [!NOTE]
> `GuildId` を指定して登録したあとにグローバル登録へ切り替えると、そのサーバーではコマンドが重複して表示されます。

### 3. 起動する

```sh
dotnet run
```

ログに `Logged in as ...` と `Registered slash commands ...` が出れば起動完了です。

## 開発

### コマンドの追加

[Modules/](Modules/) に `InteractionModuleBase<SocketInteractionContext>` を継承した `public` クラスを追加するだけで、起動時に自動で読み込まれます。

```cs
using Discord.Interactions;

namespace sard_discord_bot.Modules;

public class PingModule : InteractionModuleBase<SocketInteractionContext>
{
    [SlashCommand("ping", "応答を返します")]
    public async Task PingAsync() => await RespondAsync("pong");
}
```

- クラスが `public` でないと、エラーにならずに無視されます。
- モジュールはコマンドが呼ばれるたびに新しく作られます。状態を保持したい場合は、シングルトンのサービスを [Program.cs](Program.cs) で登録し、コンストラクタで受け取ってください。
- コマンドを追加・変更したら、必要な権限を [RequiredPermission.md](RequiredPermission.md) に追記してください。

### 構成

| ファイル | 役割 |
|---|---|
| [Program.cs](Program.cs) | DI 登録と Gateway Intents の設定 |
| [Worker.cs](Worker.cs) | ログイン、コマンドの登録、インタラクションの処理 |
| [Modules/](Modules/) | スラッシュコマンドの実装 |
| [RequiredPermission.md](RequiredPermission.md) | 必要な Scope・権限・Intent の一覧 |

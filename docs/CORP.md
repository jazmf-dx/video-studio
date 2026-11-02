# 社内版

jazmf-dx/video-studio でだけ扱う、会社の PC（主に Windows）向けのビルド。中身は hamirilo/video-studio（upstream）と同じコードで、重ねる設定ファイル（`src-tauri/tauri.corp.conf.json`）で名前・ID・更新の配信先・署名の鍵を社内版のものにしている。

| | 個人版（upstream） | 社内版 |
|---|---|---|
| 機能 | すべて | 同じ（外している機能はない） |
| アプリ内アップデート | hamirilo/video-studio のリリースから | jazmf-dx/video-studio のリリースから。社内版専用の鍵で署名し、検査を通して Publish した版だけが届く |
| 製品名・ID | `Video Studio` / `dev.workbench.video-studio` | `Video Studio Corp` / `dev.workbench.video-studio.corp` |
| インストーラ | macOS の .dmg、Windows の -setup.exe | Windows の -setup.exe だけ |

ID が違うので、設定・キャッシュ・ログなどアプリのフォルダは個人版とは別になる。

## 守ること：upstream と同じファイルは変えない

jazmf-dx は「upstream ＋ 社内版だけのファイル」にしておく。upstream にあるファイル（`src/`・`src-tauri/src/`・`tauri.conf.json`・`README.md`・`CLAUDE.md`・`build.yml` など）を jazmf-dx で直接変えると、upstream がそのファイルを変えるたびに取り込みで衝突し、差がだんだん広がっていく。

| やりたいこと | どこで、どうやるか |
|---|---|
| 機能の改修・バグ修正（社内で見つけたものも含む） | upstream で直して、取り込む |
| 社内版で機能を外したい | まず upstream に、外せる仕組みを汎用の形で足す（stockroom の `docs/LIMITS.md` と同じ考え方）。社内版では `corp.yml` の設定で使うだけにする |
| 製品名・ID・インストーラ・配信先・公開鍵 | `src-tauri/tauri.corp.conf.json` |
| 社内の運用の手順 | この文書 |

社内版にしか要らない変更も、コードに「社内版なら」と書くのではなく、「この機能を外せる」という形で upstream に入れる。更新を確かめるときのプロキシの判定と「更新内容を見る」のページは、コードが `plugins.updater.endpoints` の 1 つ目のリポジトリと、ビルドのときの `VIDEO_STUDIO_TAG_PREFIX`（社内版は `corp-`）から決める（`src-tauri/src/feed.rs`）。

`main` への PR では `fork-guard` ワークフローが、upstream との差が下の「社内版だけのファイル」に収まっているかを確かめ、ほかのファイルが変わっていたら失敗する。

GitHub の画面に出る「This branch is N commits ahead of hamirilo/video-studio:main」は、社内版のコミットと取り込みのマージの数なので、増えていって問題ない。「Contribute」から upstream への PR は作らない（社内版のファイルが upstream に入ってしまう）。

## 社内版だけのファイル

- `src-tauri/tauri.corp.conf.json`：製品名・ID・インストーラの種類（NSIS だけ）を決める。updater の配信先と公開鍵を社内版のものにする
- `.github/workflows/corp.yml`：tag `corp-v<版>` で、テストとビルドをして、下書きのリリースを作る
- `.github/workflows/fork-guard.yml`：PR で、upstream との差がこれらのファイルだけかを確かめる
- `docs/CORP.md`：この文書

## 開発

```bash
bun run ffmpeg
VIDEO_STUDIO_TAG_PREFIX=corp- bun run tauri dev --config src-tauri/tauri.corp.conf.json
```

## upstream の取り込み

```bash
git remote add upstream https://github.com/hamirilo/video-studio   # 初回だけ
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

GitHub の画面の「Sync fork」でもよい。社内版だけのファイルは upstream にないので、通常は衝突しない。取り込んだあとで `git diff upstream/main main --stat` を見ると、上の「社内版だけのファイル」だけが出る。

### 個人版のワークフロー

upstream から来る `.github/workflows/build.yml`（個人版のテストとリリース）は、jazmf-dx では使わない。個人版の更新ファイルに署名する鍵を会社のリポジトリに置かないので、`main` への push でビルドが重く回るだけになる。ファイルを変えると取り込みのたびに衝突するので、GitHub の画面で止める：Actions → build →「…」→ **Disable workflow**。

## 署名の鍵

アプリ内アップデートは、社内版専用の鍵で署名した更新しか受け付けない。公開鍵は `tauri.corp.conf.json` の `plugins.updater.pubkey`。個人版の鍵とは別のもので、個人版の更新が社内版に入ることはない（逆も同じ）。jazmf-dx/desktop-apps の社内版と同じ鍵を使ってよい。

```bash
bunx @tauri-apps/cli signer generate -w video-studio-corp.key   # desktop-apps と共有するなら、そちらの鍵を使う
```

- 公開鍵を `tauri.corp.conf.json` の `pubkey`（`REPLACE_WITH_CORP_PUBLIC_KEY`）に入れて、PR で `main` に入れる。入れるまでは `corp` ワークフローが止まる
- 秘密鍵とパスワードは、Environment `corp-release` の Secret（`CORP_SIGNING_PRIVATE_KEY`・`CORP_SIGNING_PRIVATE_KEY_PASSWORD`）にだけ置く。Repository secret にはしない
- `corp-release` は、tag `corp-v*` からしか使えないようにする（Settings → Environments → Deployment branches and tags）
- 秘密鍵のファイルは手元に残さないか、パスワード管理ツールなど安全な場所に置く。漏れたら、その鍵で署名した更新が全員の PC に入りうる
- 鍵をなくしたり作り直したりすると、それまでの社内版には更新が届かなくなる

## リリース

1. upstream を取り込む（上の手順）
2. 個人版と同じ番号でタグを付ける：`git tag corp-vX.Y.Z && git push origin corp-vX.Y.Z`
3. GitHub Actions の `corp` ワークフローがテストし、Windows のインストーラを作って、**下書きのリリース** `corp-vX.Y.Z` に添付する（`-setup.exe`・`.sig`・`latest.json`。ハッシュは実行のサマリーと Artifacts の `SHA256SUMS.txt`）。下書きのうちは、アプリ内アップデートからは見えない
4. 下書きのリリースから `-setup.exe` を取り、検査に提出する。そのときに SHA-256 を控える
5. 検査を通ったら、Publish する前に、下書きに添付されている `-setup.exe` の SHA-256 が控えと同じか確かめる。同じタグでワークフローを動かし直すと添付が差し替わるので、検査のあとは動かし直さない

   ```bash
   gh release download corp-vX.Y.Z -R jazmf-dx/video-studio -p '*-setup.exe' -D check
   sha256sum check/*-setup.exe   # Windows: Get-FileHash check\*-setup.exe
   ```

6. 下書きの本文に変更点を書き、**Publish** する。この時点で、社内版を使っている全員のアプリに更新が届く
7. まだ社内版を入れていない人には、同じ `-setup.exe` を社内の共有の場所に置いて配る

Publish したあとで問題が見つかったら、リリースを下書きに戻す（Edit → Save as draft）。それ以降の配信は止まるが、すでに更新した PC は戻らない。直した版を、次の番号で出す。

### 版の番号

社内版の番号は、取り込んだ upstream の番号と同じにする（`tauri.conf.json` は jazmf-dx では変えないので、自然に揃う）。間の版は飛ばしてよい。

`tauri.corp.conf.json` や `corp.yml` だけを変えて出し直すときは、upstream の番号が上がるのを待つ（急ぐなら upstream でパッチ版を出す）。急ぎで番号を上げられないときはタグに枝番を付ける（`corp-v0.1.3-2`）。アプリ内アップデートでは届かないので手で入れ替えてもらい、Publish もしない。

インストーラはコード署名をしていないので、初回は SmartScreen の警告が出る。

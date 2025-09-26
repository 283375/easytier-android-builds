# EasyTier 自定义构建

结果在 [Releases](https://github.com/283375/easytier-android-builds/releases) 中。

## 更改项

### 启用 split abi

主要修改于构建 workflow，详见 [build-apk.yml](.github/workflows/build-apk.yml)

<details>

<summary><i>git diff --no-index mobile.yml .github/workflows/build-apk.yml</i></summary>

```diff
diff --git a/mobile.yml b/.github/workflows/build-apk.yml
index 102a5c5..9114472 100644
--- a/mobile.yml
+++ b/.github/workflows/build-apk.yml
@@ -1,53 +1,35 @@
-name: EasyTier Mobile
+name: Build EasyTier Android APK
 
 on:
-  push:
-    branches: ["develop", "main", "releases/**"]
-  pull_request:
-    branches: ["develop", "main"]
+  workflow_call:
+    inputs:
+      source_artifact_name:
+        description: "The artifact name of patched source code"
+        required: false
+        type: string
+        default: source-patched
+    outputs:
+      outputs_artifact_name:
+        description: "The artifact name of built APKs"
+        value: "easytier-gui-android"
 
 env:
   CARGO_TERM_COLOR: always
 
-defaults:
-  run:
-    # necessary for windows
-    shell: bash
-
 jobs:
-  pre_job:
-    # continue-on-error: true # Uncomment once integration is finished
-    runs-on: ubuntu-latest
-    # Map a step output to a job output
-    outputs:
-      should_skip: ${{ steps.skip_check.outputs.should_skip == 'true' && !startsWith(github.ref_name, 'releases/') }}
-    steps:
-      - id: skip_check
-        uses: fkirc/skip-duplicate-actions@v5
-        with:
-          # All of these options are optional, so you can remove them if you are happy with the defaults
-          concurrent_skipping: "same_content_newer"
-          skip_after_successful_duplicate: "true"
-          cancel_others: "true"
-          paths: '["Cargo.toml", "Cargo.lock", "easytier/**", "easytier-gui/**", "tauri-plugin-vpnservice/**", ".github/workflows/mobile.yml", ".github/workflows/install_rust.sh"]'
-  build-mobile:
-    strategy:
-      fail-fast: false
-      matrix:
-        include:
-          - TARGET: android
-            OS: ubuntu-22.04
-            ARTIFACT_NAME: android
-    runs-on: ${{ matrix.OS }}
+  build:
+    name: Build
     env:
+      OS: ubuntu-latest
       NAME: easytier
-      TARGET: ${{ matrix.TARGET }}
-      OS: ${{ matrix.OS }}
-      OSS_BUCKET: ${{ secrets.ALIYUN_OSS_BUCKET }}
-    needs: pre_job
-    if: needs.pre_job.outputs.should_skip != 'true'
+      TARGET: android
+    runs-on: ubuntu-latest
+
     steps:
-      - uses: actions/checkout@v3
+      - name: Fetch source code
+        uses: actions/download-artifact@v5
+        with:
+          name: ${{ inputs.source_artifact_name }}
 
       - name: Set current ref as env variable
         run: |
@@ -68,11 +50,7 @@ jobs:
         run: |
           echo "$ANDROID_HOME/platform-tools" >> $GITHUB_PATH
           echo "$ANDROID_HOME/ndk/26.0.10792818/toolchains/llvm/prebuilt/linux-x86_64/bin" >> $GITHUB_PATH
-          echo "NDK_HOME=$ANDROID_HOME/ndk/26.0.10792818/" > $GITHUB_ENV
-
-      - uses: actions/setup-node@v4
-        with:
-          node-version: 22
+          echo "NDK_HOME=$ANDROID_HOME/ndk/26.0.10792818/" >> $GITHUB_ENV
 
       - name: Install pnpm
         uses: pnpm/action-setup@v4
@@ -80,18 +58,11 @@ jobs:
           version: 10
           run_install: false
 
-      - name: Get pnpm store directory
-        shell: bash
-        run: |
-          echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_ENV
-
-      - name: Setup pnpm cache
-        uses: actions/cache@v4
+      - name: Install Node.js
+        uses: actions/setup-node@v4
         with:
-          path: ${{ env.STORE_PATH }}
-          key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
-          restore-keys: |
-            ${{ runner.os }}-pnpm-store-
+          node-version: 22
+          cache: pnpm
 
       - name: Install frontend dependencies
         run: |
@@ -121,12 +92,16 @@ jobs:
       - name: Build Android
         run: |
           cd easytier-gui
-          pnpm tauri android build
+          pnpm tauri android build --apk --split-per-abi
+
+      - name: Check outputs
+        run: |
+          tree easytier-gui/src-tauri/gen/android/app/build/outputs/apk
 
-      - name: Compress
+      - name: Prepare artifacts
         run: |
-          mkdir -p ./artifacts/objects/
-          mv easytier-gui/src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release.apk ./artifacts/objects/
+          mkdir -p ./artifacts
+          find . -type f -name "*.apk" -print0 | xargs -0 -I {} mv {} ./artifacts/
 
           if [[ $GITHUB_REF_TYPE =~ ^tag$ ]]; then
             TAG=$GITHUB_REF_NAME
@@ -134,23 +109,10 @@ jobs:
             TAG=$GITHUB_SHA
           fi
 
-          mv ./artifacts/objects/* ./artifacts
-          rm -rf ./artifacts/objects/
-
-      - name: Archive artifact
+      - name: Upload artifacts
         uses: actions/upload-artifact@v4
         with:
-          name: easytier-gui-${{ matrix.ARTIFACT_NAME }}
+          name: easytier-gui-android
           path: |
             ./artifacts/*
-
-  mobile-result:
-    if: needs.pre_job.outputs.should_skip != 'true' && always()
-    runs-on: ubuntu-latest
-    needs:
-      - pre_job
-      - build-mobile
-    steps:
-      - name: Mark result as failed
-        if: needs.build-mobile.result != 'success'
-        run: exit 1
+        id: upload-artifacts
```

</details>

### 图标与应用 ID 更改

图标源文件见 [app-icon-tauri.svg](./app-icon-tauri.svg)

## 主要构建流程

> 详见 [main.yml](.github/workflows/main.yml) 的 `prepare` 部分

1. 拉取 [EasyTier/EasyTier](https://github.com/EasyTier/EasyTier) 源码
2. Apply `*.patch`es
3. 删掉 `.git` 和其他无用文件后打包存入 `/tmp`
4. 替换工作目录为此 repo 的 [app](https://github.com/283375/easytier-android-builds/tree/app) 分支
5. 将 3. 中的包解压覆盖
6. `git add . && git commit && git tag`
7. 构建 & Release & ...
8. ...
9. bang15便士
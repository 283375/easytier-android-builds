# Custom Android Builds

## Build Workflow Diff

> git diff --no-index mobile.yml .github/workflows/build-apk.yml

```diff
diff --git a/mobile.yml b/.github/workflows/build-apk.yml
index 102a5c5..e483dbd 100644
--- a/mobile.yml
+++ b/.github/workflows/build-apk.yml
@@ -1,53 +1,25 @@
-name: EasyTier Mobile
+name: Build EasyTier Mobile APK
 
 on:
-  push:
-    branches: ["develop", "main", "releases/**"]
-  pull_request:
-    branches: ["develop", "main"]
+  workflow_call:
 
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
+  build:
+    name: Build
     runs-on: ubuntu-latest
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
+
     env:
       NAME: easytier
-      TARGET: ${{ matrix.TARGET }}
-      OS: ${{ matrix.OS }}
-      OSS_BUCKET: ${{ secrets.ALIYUN_OSS_BUCKET }}
-    needs: pre_job
-    if: needs.pre_job.outputs.should_skip != 'true'
+      TARGET: android
+
     steps:
-      - uses: actions/checkout@v3
+      - name: Fetch source code
+        uses: actions/download-artifact@v5
+        with:
+          name: source-patched
 
       - name: Set current ref as env variable
         run: |
@@ -70,28 +42,17 @@ jobs:
           echo "$ANDROID_HOME/ndk/26.0.10792818/toolchains/llvm/prebuilt/linux-x86_64/bin" >> $GITHUB_PATH
           echo "NDK_HOME=$ANDROID_HOME/ndk/26.0.10792818/" > $GITHUB_ENV
 
-      - uses: actions/setup-node@v4
-        with:
-          node-version: 22
-
       - name: Install pnpm
         uses: pnpm/action-setup@v4
         with:
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
@@ -121,7 +82,7 @@ jobs:
       - name: Build Android
         run: |
           cd easytier-gui
-          pnpm tauri android build
+          pnpm tauri android build --apk --split-per-abi
 
       - name: Compress
         run: |
@@ -140,17 +101,6 @@ jobs:
       - name: Archive artifact
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
```

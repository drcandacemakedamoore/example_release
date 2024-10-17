# Summary of Work Necessary to Restore Release Functionality - on setup.py
# in on-tag change upload to upload-artifact@v4

```diff
diff --git a/.github/workflows/on-tag.yml b/.github/workflows/on-tag.yml
new file mode 100644
index 0000000..53d1717
--- /dev/null
+++ b/.github/workflows/on-tag.yml
@@ -0,0 +1,98 @@
+name: Release
+
+on:
+  push:
+    tags:
+      - v*
+# on: [push]
+
+jobs:
+  PyPIBuild:
+    #if- get rid of if because repo is different anyways
+    name: Tagged Release
+    runs-on: ubuntu-latest
+    steps:
+      - uses: actions/checkout@v2
+        with:
+          fetch-depth: 0
+          submodules: recursive
+        # Unfortunately, wheel will try to do setup.py install to
+        # build a wheel... and we need this stuff to be able to build
+        # for CPython.
+      
+      - name: Set up Python 3.11
+        uses: actions/setup-python@v1
+        with:
+          python-version: 3.11
+      - run: python3.11 -m venv .venv
+      - run: .venv/bin/python -m pip install wheel twine
+      - run: .venv/bin/python setup.py bdist_wheel
+      - run: >-
+          TWINE_USERNAME=__token__
+          TWINE_PASSWORD=${{ secrets.PYPI_TOKEN }}
+          .venv/bin/python -m twine upload --repository testpypi --skip-existing ./dist/*.whl
+      - uses: actions/upload-artifact@v4
+        with:
+          name: pypi-build
+          path: dist/*
+
+  CondaBuild:
+    runs-on: ${{ matrix.os }}
+    defaults:
+      run:
+        shell: bash -el {0}
+    # defaults:
+    #   run:
+    #     shell: bash -el {0}
+    strategy:
+      fail-fast: false
+      matrix:
+        python-version: [3.11]
+        os: [ubuntu-latest, windows-latest, macos-latest]
+        include:
+          - { os: windows-latest, shell: msys2 }
+          - { os: ubuntu-latest,  shell: bash  }
+          - { os: macos-latest,   shell: bash  }
+    steps:
+      - uses: actions/checkout@v2
+        with:
+          submodules: recursive
+          fetch-depth: 0
+      - uses: conda-incubator/setup-miniconda@v3
+      - uses: conda-incubator/setup-miniconda@v3
+        with:
+          python-version: ${{ matrix.python-version }}
+          auto-activate-base: false
+          activate-environment: ci
+          channels: conda-forge
+      - run: conda config --remove channels defaults
+      - name: Generate conda meta.yaml (Python 3.11)
+        run: python -u setup.py anaconda_gen_meta
+      - run: python -u setup.py bdist_conda
+      - name: Upload Anaconda package
+        run: >-
+          python setup.py anaconda_upload
+          --token=${{ secrets.ANACONDA_TOKEN }}
+          --package=./dist/*/*.tar.bz2
+      - uses: actions/upload-artifact@v4
+        with:
+          name: conda-build-${{ matrix.os }}-${{ matrix.python-version }}
+          path: dist/*/*.tar.bz2
+
+  PublishArtifacts:
+    runs-on: ubuntu-latest
+    needs: [PyPIBuild, CondaBuild]
+    steps:
+      - uses: actions/download-artifact@v2
+        with:
+          path: dist
+      - uses: marvinpinto/action-automatic-releases@latest
+        with:
+          repo_token: "${{ secrets.GITHUBTOKEN }}"
+          prerelease: false
+          files: |
+            ./dist/*/linux-64/cvasl-*.tar.bz2
+            ./dist/*/osx-64/cvasl-*.tar.bz2
+            ./dist/*/win-64/cvasl-*.tar.bz2
+            ./dist/pypi-build/*.whl
+            ./dist/pypi-build/*.egg
diff --git a/setup.py b/setup.py
index dbe7e69..06adcf0 100644
--- a/setup.py
+++ b/setup.py
@@ -405,9 +405,9 @@ class SdistConda(Command):
         self.distribution.install_requires = translate_reqs(
             self.distribution.install_requires,
         )
-        self.distribution.tests_require = translate_reqs(
-            self.distribution.tests_require,
-        )
+        # self.distribution.tests_require = translate_reqs(
+        #     self.distribution.tests_require,
+        # )
         bdist_egg = BDistEgg(self.distribution)
         bdist_egg.initialize_options()
         bdist_egg.finalize_options()
@@ -632,7 +632,7 @@ if __name__ == '__main__':
             'umap-learn>=0.5.1',
             'yellowbrick>=1.3',
         ],
-        tests_require=['pytest', 'nbmake', 'pycodestyle', 'isort', 'wheel'],
+        # tests_require=['pytest', 'nbmake', 'pycodestyle', 'isort', 'wheel'],
         setup_requires=['wheel'],
         extras_require={
             'dev': [
```
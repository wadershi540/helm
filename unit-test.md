# Helm 可以處理非 .yaml 副檔名的檔案嗎?

https://github.com/helm/helm/issues/7899

根據討論內容，目前不支援非 yaml 檔案的渲染。
```
jdolitsky commented on Apr 14, 2020
@LvBay - currently this is not supported. Just curious - what is your intended usage of this script? In order to run within the context of Kubernetes, it would need to be in the format of a ConfigMap which Helm can template
```

不過其中有個回答，可以試著將內容改成 yaml 格式，helm template 就可以處理了。
```bash
$ cat templates/t
aaa: |
  b = {{ .Values.b | quote }}
$ cat values.yaml 
b: bbb
$ helm template . -f templates/t
---
# Source: chart-1/templates/t
aaa: |
  b = "bbb"
```

延伸問題：
* 上面 t 檔案在部屬的時候是否會有問題? 是否會被當成 k8s resource 部屬?
* t 檔案是否可以拿來執行 unit test

# Helm Unit Test 無法測試非 .yaml 結尾的檔案

根據下面測試結果，雖然 `helm template` 可以渲染 `templates/t` 檔案，但實際上 unit test 卻無法針對 `templates/t` 做測試。

```bash
$ ls templates/t
templates/t
$ cat tests/t_test.yaml 
templates:
  - templates/t
tests:
  - it: should pass
    asserts:
      - isSubset:
          path: aaa
          content:
            b = bbb
$ helm unittest .

### Chart [ chart-1 ] .

 FAIL   tests/t_test.yaml
        - should pass

                - asserts[0] `isSubset` fail
                        Error:
                                template "chart-1/templates/t" not exists or not selected in test suite


Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 0 passed, 1 total
Snapshot:    0 passed, 0 total
Time:        2.044841ms

Error: plugin "unittest" exited with error
```

思考還可以怎麼做?
* 在 templates 底下寫 yaml, 尋找可以 bypass 特定 yaml deploy 到 k8s
* 使用 subchart 做測試?

# 透過 .helmignore bypass 特定 YAML file 進行 k8s 部屬，是否可以達到 unit test

不行!

雖然 .helmignore 可以 bypass 特定 YAML 進行部屬 (實際上我只有驗證到 `helm template` 可以 bypass 特定 YAML，但應該會 bypass)，但是在 `helm unittest` 過程中，也找不到特定 YAML 進行 unit test.

```bash
$ cat .helmignore 
...
templates/t.yaml
$ cat tests/t_test.yaml 
templates:
  - templates/t.yaml
tests:
  - it: should pass
    asserts:
      - isSubset:
          path: aaa
          content:
            aaa: |
              b = "bbb"
$ helm unittest .

### Chart [ chart-1 ] .

 FAIL   tests/t_test.yaml
        - should pass

                - asserts[0] `isSubset` fail
                        Error:
                                template "chart-1/templates/t.yaml" not exists or not selected in test suite


Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 0 passed, 1 total
Snapshot:    0 passed, 0 total
Time:        1.74293ms

Error: plugin "unittest" exited with error
```

將 `templates/t.yaml` 從 .helmignore 移除後，就可以正常 unit test 了。
```bash
$ cat .helmignore 
...
#templates/t.yaml
$ helm unittest .

### Chart [ chart-1 ] .

 PASS   tests/t_test.yaml

Charts:      1 passed, 1 total
Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshot:    0 passed, 0 total
Time:        4.640478ms
```

找不到其它可以 bypass YAML 部屬，又可以同時做 unit test 的方法，接下來試試看 subchart. 將 unit test 的檔案放到 subchart (`subchart/templates/some_files.yaml`), 把 subchart 當成 unit test 用途，並且不要將其引入到 parent chart 是否可以?

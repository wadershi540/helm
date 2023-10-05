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

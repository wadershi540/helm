```
chart
├── main
│   ├── Chart.yaml
│   └── values.yaml
└── sub1
    ├── Chart.yaml
    ├── templates
    │   └── configmap.yaml
    └── values.yaml
```

```
# chart\main\Chart.yaml
...
dependencies:
  - name: sub1
    version: 0.1.0
    repository: "file://../sub1"
```

根據 helm dependency 的寫法，執行 helm template 遇到以下錯誤

```
$ helm template .
Error: An error occurred while checking for chart dependencies. You may need to run `helm dependency build` to fetch missing dependencies: found in Chart.yaml, but missing in charts/ directory: sub1
```

相關指令 check & fix，透過 helm dependency build 會將 file reference 的 chart 目錄，產生 lock file & tar file
```
$ helm dep ls
NAME    VERSION REPOSITORY      STATUS
sub1    0.1.0   file://../sub1  missing

$ helm dependency build
Saving 1 charts
Deleting outdated charts

$ helm dep ls
NAME    VERSION REPOSITORY      STATUS
sub1    0.1.0   file://../sub1  ok

# main folder tree
main
├── Chart.lock
├── Chart.yaml
├── charts
│   └── sub1-0.1.0.tgz
└── values.yaml
```

透過 helm template 看看 sub1/template/configmap.yaml 產生的內容是什麼?
```
$ helm template .
---
# Source: main/charts/sub1/templates/configmap.yaml
common: sub1
```

如果要將 main/Values.yaml 底下的 value 帶進 sub1/template/confugmap.yaml，必須作對應修改如下
```
# chart\main\Chart.yaml
dependencies:
  - name: sub1
    version: 0.1.0
    repository: "file://../sub1"
    alias: sub1 # 多新增 alias 名稱，可任意取

# chart\main\values.yaml
sub1: # 這邊的 key name 必須跟上面的 alias 名稱一致
  common: main
```

在一次執行 helm template 可以看到 configmap.yaml 裡面參考的 value 是來自於 main/values.yaml
```
# chart\main\values.yaml
sub1:
  common: main

# chart\sub1\values.yaml
common: sub1

# chart\sub1\templates\configmap.yaml
common: {{ .Values.common }}

$ helm template .
---
# Source: main/charts/sub1/templates/configmap.yaml
common: main
```
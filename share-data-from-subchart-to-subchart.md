# Sharing Data Between Subcharts

- [Sharing Data Between Subcharts](#sharing-data-between-subcharts)
  - [How](#how)
  - [問題](#問題)
  - [Solution 1: 透過 global values](#solution-1-透過-global-values)
    - [Step 1: 將共同部份抽出，放到 common chart 的 values.yaml](#step-1-將共同部份抽出放到-common-chart-的-valuesyaml)
    - [Step 2: main chart depend with common chart](#step-2-main-chart-depend-with-common-chart)
    - [Output](#output)
  - [Solution 2: 透過 import-values 到相同的 subchart name](#solution-2-透過-import-values-到相同的-subchart-name)
    - [Step 1: 一樣將共同部份抽出，放到 common chart 的 values.yaml](#step-1-一樣將共同部份抽出放到-common-chart-的-valuesyaml)
    - [Step 2: main chart depend with common chart using subchart name](#step-2-main-chart-depend-with-common-chart-using-subchart-name)
    - [Output for solution 2](#output-for-solution-2)

## How

可以透過 parent chart `Chart.yaml` 的 `import-values` 來完成，其中有兩種方式會影響 value 的 priority。

## 問題

有一個 main chart，主要用來做 deploy 用的，它的用途只是提供 values.yaml 給 sub1 這個 chart。

假設在某個部屬環境， main1~9 這些 chart 全部都提供 values.yaml 然後部屬 sub1 chart，但 main1~9 底下的 values.yaml 可能有共用的部分，要如何抽出這些共同的部分成為一個單一檔案。

## Solution 1: 透過 global values

將 values.yaml 共同的部分，抽出來存放到 common chart，透過 main1~9 Chart.yaml 裡面宣告 dependencies common chart。

### Step 1: 將共同部份抽出，放到 common chart 的 values.yaml

```yaml
# chart\common\values.yaml
by_import_global: # by_import_global 是整個用被共用的 root data，名稱可隨意取，但在 import-values 要上對應的名稱
  # data01: 
  # data02:
  data03: common.data03
  data04: common.data04
  data05: common.data05
  # data06:
  data07: common.data07
```

### Step 2: main chart depend with common chart

在 main chart 的 Chart.yaml 寫入 common chart dependency，並透過 `import-values` 的方式將要共用的部分傳遞給 parent chart 的 `global` 變數。

```bash
# chart\main\Chart.yaml
...
dependencies:
  - name: common
    version: 0.1.0
    repository: "file://../common"
    import-values:
      - child: by_import_global
        parent: global # 重點
...
```

### Output

觀察 template 後的結果

```bash
$ helm dep update; helm template .
Saving 2 charts
Deleting outdated charts
---
# Source: main/charts/sub1/templates/configmap.yaml
# main.data01: main.data01
# sub1.data01:
# common.data01:
data01: main.data01

# main.data02:
# sub1.data02: sub1.data02
# common.data02:
data02: sub1.data02

# main.data03:
# sub1.data03:
# common.data03: common.data03
data03: common.data03

# main.data04:
# sub1.data04: sub1.data04
# common.data04: common.data04
data04: common.data04

# main.data05: main.data05
# sub1.data05:
# common.data05: common.data05
data05: main.data05

# main.data06: main.data06
# sub1.data06: sub1.data06
# common.data06:
data06: main.data06

# main.data07: main.data07
# sub1.data07: sub1.data07
# common.data07: common.data07
data07: main.data07
```

從輸出結果可以看到 values 的優先順序是 main > common > subchart。

## Solution 2: 透過 import-values 到相同的 subchart name

將 values.yaml 共同的部分，抽出來存放到 common chart，透過 main1~9 Chart.yaml 裡面宣告 dependencies common chart。

### Step 1: 一樣將共同部份抽出，放到 common chart 的 values.yaml

```yaml
# chart\common\values.yaml
by_import:
  # data11: 
  # data12:
  data13: common.data13
  data14: common.data14
  data15: common.data15
  # data16:
  data17: common.data17
```

### Step 2: main chart depend with common chart using subchart name

跟 solution 1 一樣，但這次是 import 到 parent chart 的 `sub1` 這個變數，這個變數名稱必須跟 subchart 要用的變數名稱一樣。

```bash
# chart\main\Chart.yaml
...
dependencies:
  - name: common
    version: 0.1.0
    repository: "file://../common"
    import-values:
      - child: by_import
        parent: sub1 # 重點
...
  - name: sub1
    version: 0.1.0
    repository: "file://../sub1"
    alias: sub1 # 多新增 alias 名稱，可任意取

```

### Output for solution 2

觀察 template 後的結果

```bash
$ helm dep update; helm template .
Saving 2 charts
Deleting outdated charts
---
# main.data11: main.data11
# sub1.data11:
# common.data11:
data11: main.data11

# main.data12:
# sub1.data12: sub1.data12
# common.data12:
data12: sub1.data12

# main.data13:
# sub1.data13:
# common.data13: common.data13
data13: common.data13

# main.data14:
# sub1.data14: sub1.data14
# common.data14: common.data14
data14: sub1.data14

# main.data15: main.data15
# sub1.data15:
# common.data15: common.data15
data15: main.data15

# main.data16: main.data16
# sub1.data16: sub1.data16
# common.data16:
data16: main.data16

# main.data17: main.data17
# sub1.data17: sub1.data17
# common.data17: common.data17
data17: main.data17
```

從輸出結果可以看到 values, 如 data14 & data17 的優先順序是 main > subchart > common。

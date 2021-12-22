class: center, middle

## DDD Study Group
# Week 3 - Context Maps

2021/12/23 @ Hahow

---

# Agenda

1. 如何參與
2. 一分鐘 Recap
3. 以 2B 為例的 Context Map
4. 分組討論: 一起來畫 Context Map 吧

---

# 1. 如何參與

- 對大家會有幫助的問題，請隨時打斷問！
- 其他問題可在 Slack 或聊天室問
- 這是讀書會，目標是互動為主、講者為輔 XD
- 如果有發現講得怪怪的地地方，請務必隨時打斷我一起釐清！

---

# 2. 一分鐘 Recap

### Prerequisite

請區分 Domain 跟 Bounded Context？

### IDDD Ch3 Context Maps（預設各位已讀 :troll:）


<img width="30%" src=https://user-images.githubusercontent.com/12410942/146681659-d82703a2-7eb0-4adb-84cb-2b7c3e80fbba.png />
<img width="30%" src=https://user-images.githubusercontent.com/12410942/146681878-51ef0f38-9e4a-44db-8a1e-ce8e795eadb6.jpg />

互動： 有讀的請喊 Y

???

- Domain: 對應一套 Ubiquitous Language
- Bounded Context: 軟體邊界，理想上與 Domain 一一對應

---

# Chapter Overview (bonus)

### IDDD Ch 13 Integrating Bounded Contexts

Ch3 點到了「程式面」的整合方式（e.g. Translator）

Ch13 繼續介紹「系統面」的整合方式，挑了 RESTful API 跟 Message Queue 這兩種方式介紹（的樣子）

---

# Chapter Overview (bonus)

### DDD Ch 14

從 Bounded Context 講起，**詳述** Context Maps 與 7 種 relationship（所以 IDDD 才這麼精簡？）

<img width="30%" src=https://user-images.githubusercontent.com/12410942/146681461-346dae5e-0fdc-45a9-9216-26b80ffbe6d1.jpg />

---

name: relationship-list

3. 以 2B 為例的 Context Map

- Shared Kernel
- Separate Way
- Customer/Supplier Development Team
- Conformist
- Anticorruption Layer
- Open Host Service
- Published Language

⚠️ 大量使用 2B/2C 合作案例舉例，請勿走心，大家都很棒，一切都是 trade off，一起繼續努力 ❤️

---

## Shared Kernel v.s. Separate Way

- 期許有朝一日的 [vod-service](https://github.com/hahow/vod-service)
- 曾經理想中的 [hh-classroom](https://github.com/hahow/hh-classroom/)

<img style="width: 50%;" src=https://user-images.githubusercontent.com/559351/65013811-2bae5580-d94f-11e9-995a-136c29f8fd3a.png />


其實就是共用與否的權衡，Shared Kernel 的成功需要：
1. 有意識共用 Ubiquitous Language 並一起維護（or 轉換層的成本多大？）
2. Continuous Integration（修改不能 break 彼此）

---

## Conformist

2B 2C 合作的 worse case：上游團隊沒有動力優先滿足下游團隊的需求
<img width="50%" src="https://user-images.githubusercontent.com/12410942/146956198-8fead608-9691-48dc-b061-1fd98880a298.png">

根據不同情況...

情境 1: 高估了依賴的價值，低估了成本 👉 Separate Way

情境 2: 上游的設計封裝不良 👉 下游團隊負責維護一個 Anticorruption Layer

情境 3: 上游的設計沒問題，但概念不同 👉 下游團隊負責維護一個 Anticorruption Layer / Translator

情境 4: 上游團隊的設計不錯 👉 下游直接當 Conformist

---

### 情境 1: 高估了依賴的價值，低估了成本 👉 Separate Way

hh-backend 課程匯入 hahow-for-business
`POST /courses/import` 直接複製一份
<img width=100% src="https://user-images.githubusercontent.com/12410942/146716322-b37ec41f-2be3-4500-b789-a63998571116.png">

---

### 情境 2: 上游的設計封裝不良或太複雜 👉 下游團隊負責維護一個 Anticorruption Layer

- 把綠界封裝起來

<img width="100%" src="https://user-images.githubusercontent.com/12410942/146958726-e49e9094-efa7-4460-b70d-8c6594f95084.png">

---

### 情境 3: 上游的設計沒問題，但概念不同 👉 下游團隊負責維護一個 Anticorruption Layer / Translator

- iOS 有 anti corruption layer 把 API 的 resource 轉換成對 iOS app 最直覺的 domain model
- hahow-for-business 把 BigQuery 封裝在薄薄的轉換層後面
<img width="100%" src="https://user-images.githubusercontent.com/12410942/146959004-e183a62d-5209-4a7c-a29e-0f8aa9dd05e3.png">

---

### 情境 3: 上游團隊的設計不錯 👉 下游直接當 Conformist

hahow-for-business 的前端直接當 conformist 接受所有後端 GraphQL API 的命名與抽象，可大量共用 typing 減少 boilerplate

P.s. 某個元件的介面很大時可考慮 Conformist，因為該元件的 model 可能比你理解的還完整、有體系

---

## 小節

看完感覺都是 common sense？ 同感，覺得 Strategy Pattern 有一點商學院味，但其價值在於：
1. 提出一套框架與 pattern 名稱，讓討論有所依據
2. 未來整合時可以有意識的逐一盤點可能的 pattern（相較於依賴經驗的直覺）
3. 命名 convention 可參考 pattern 名稱（如：ACL的translation map）
4. 其他？

---

## 小節

<img width="90%" src="https://user-images.githubusercontent.com/12410942/147099262-df8ce05d-5bc0-4020-8602-9c3f2e05651d.png">

---

# 4. 一起來畫 Context Map 吧

https://jamboard.google.com/d/1AwNcIy0ouOboNQhYk8pLWpTJ8P1-R2dadEYf1hxLyDI/edit?usp=sharing

參考步驟：
1. 先用不同的線圈出 Subdomain（大家多討論）跟 Bounded Context（可能就是所有的 services）
2. 把 Bounded Context 之間的依賴關係拉出箭頭
3. 想想兩者個關係比較像[前述](#relationship-list)哪一種呢？標在箭頭上吧

---
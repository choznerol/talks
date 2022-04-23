
## B2B SaaS Multitenancy
### a Hahow for Business case study

<small>

- 2022/04/20: Hahow `#dev-sharing`
- 2022/04/27: AppWorks School - system design study group

</small>


<small> Slide deck: https://choznerol.github.io/talks/2022-04-27-multitenancy / [`source.md`](https://github.com/choznerol/talks/blob/main/2022-04-27-multitenancy/source.md) </small>
<br/>
<br/>

<small> 💡 Navigate with the <kbd>space</kbd> key </small>
<!-- .element: class="fragment fade-left" -->

---
---

## Agenda

1. What is multitenancy? <!-- .element: class="fragment" -->
2. Multitenancy of Hahow for Business <!-- .element: class="fragment" -->
   - Past: 1 DB
   - Present: 1 DB, N schemas
   - (Possible) Future: M DB, N schema
   - Conclusion
3. Comparison <!-- .element: class="fragment" -->
4. Q & A <!-- .element: class="fragment" -->

<small>

- 💡 Feel free to interupt when you have a question <!-- .element: class="fragment" -->
- 💡 Feel free to interupt when you have something to share <!-- .element: class="fragment" -->

</small>

<small> Slide deck: https://choznerol.github.io/talks/2022-04-27-multitenancy / [`source.md`](https://github.com/choznerol/talks/blob/main/2022-04-27-multitenancy/source.md) </small>

---
---

## 1. What is multitenancy?

1 deployment to serve N isolated "tenant"
[![IBM Technology - Multitenancy Explained](https://user-images.githubusercontent.com/12410942/163659804-db9a58a9-cc90-44e7-8794-8fd873760bca.png)][1]

[IBM Technology - Multitenancy Explained][1]

[1]: https://youtu.be/60ccSmOxpMw

---

## Objectives of multitenancy

- Covered in this talk: <!-- .element: class="fragment" -->
  - Data isolation: Database, Storage, Cache
- Beyond this talk: <!-- .element: class="fragment" -->
  - Resource isolation: CPU, Memory, Network
  - Horizontal scalability

---
---

## 2. Multitenancy of Hahow for Business

---

### Conclusion (sneak peek)

|  | 1DB | 1DB<br>N schemas | N DB<br>M schemas |
| --- | --- | --- | --- |
| Operation | ? | ? | ? |
| Multitenancy | ? | ? | ? |
| Logical Isolation | ? | ? | ? |
| Physical Isolation | ? | ? | ? |
| Scale out | ? | ? | ? |

---

### Application model

An `Organization` is a tenant
<!-- .element: class="fragment" -->

`Organization` has many `User`s
<!-- .element: class="fragment" -->

`User` has many `Enrollment`s
<!-- .element: class="fragment" -->

`Enrollment` belongs to `Course` and `User`
<!-- .element: class="fragment" -->

Private tenant data: `Enrollment`
<!-- .element: class="fragment" -->

Public data accross tenants: `Course`
<!-- .element: class="fragment" -->

❓Questions?
<!-- .element: class="fragment" -->

---
---

## 2-1. Past: single database

<div class="left">

- 🌐 Application <!-- .element: class="fragment" data-fragment-index="1" -->
  - 🛢 Databases
    - 🗂 Tables

</div>
<div class="right" style="font-size: 50%">
<div>

`users` <!-- .element: class="fragment" data-fragment-index="2" -->
| | id  | **organization_id** |
| --- | --- | --- |
| | 42<!-- .element: class="fragment highlight-green" data-fragment-index="3" --> | 3<!-- .element: class="fragment highlight-green" data-fragment-index="3" --> |
| | 43<!-- .element: class="fragment highlight-green" data-fragment-index="3" --> | 3<!-- .element: class="fragment highlight-green" data-fragment-index="3" --> |
| | 44<!-- .element: class="fragment highlight-blue" data-fragment-index="3" --> | 4<!-- .element: class="fragment highlight-blue" data-fragment-index="3" --> |
<!-- .element: class="fragment" data-fragment-index="2" -->

`enrollments` <!-- .element: class="fragment" data-fragment-index="2" -->
| | id  | user_id | course_id |
| --- | --- | --- | --- |
| | 1110<!-- .element: class="fragment highlight-green" data-fragment-index="3" -->  | 42<!-- .element: class="fragment highlight-green" data-fragment-index="3" -->  | 90 |
| | 1111<!-- .element: class="fragment highlight-blue" data-fragment-index="3" -->  | 44<!-- .element: class="fragment highlight-blue" data-fragment-index="3" -->  | 84 |
| | 1112<!-- .element: class="fragment highlight-green" data-fragment-index="3" -->  | 43<!-- .element: class="fragment highlight-green" data-fragment-index="3" -->  | 101 |
<!-- .element: class="fragment" data-fragment-index="2" -->

</div>
</div>

---

### Implementation

```ruby [1|3-7|9-10|12-13|15-16|18-19]
# 💡 Feature#1: List activated Cathay users

# ✅ Correct
User.where(
  organization_id: current_organization.id,
  activated: true
).all

# ❌ Bug! Data Leakage
User.where(activated: true)

# It can become hard to debug soon ...
# 💡 Feature#2: List enrollments of activated Cathay users

# ✅ Correct
Enrollment.where(user: User.of_current_organization.activated)

# ❌ Bug! Data Leakage
Enrollment.where(user: User.activated)
```

❓How would you solve this<!-- .element: class="fragment" --> **"leakage by default"** <!-- .element: class="fragment" -->

---


## Conclusion

|  | <small>Past:<br>1DB</small> | <small>Present:<br>1DB, N-schemas</small> | <small>Future:<br>N-DB, M-schemas |
| --- | --- | --- | --- |
| Operation | low | ? | ? |
| Multitenancy | ✅ | ? | ? |
| Logical Isolation |  | ? | ? |
| Physical Isolation |  | ? | ? |
| Scale out |  | ? | ? |

---
---

## 2-2. Present: 1 database, N schemas

---

### Data isolation (1/4) - Database

<div class="left">

- 🌐 Application <!-- .element: class="fragment" data-fragment-index="1" -->
  - 🛢 Databases
    -  📂 **Schemas**
       - 🗂 Tables

<small class="fragment" data-fragment-index="1">

- [PostgreSQL Schema](https://www.postgresql.org/docs/current/ddl-schemas.html) is like "namespace" for tables
- `database.schema.table` <!-- .element: class="fragment" data-fragment-index="2" -->
  - e.g.<!-- .element: class="fragment" data-fragment-index="2" --> `hfb_prod.tenant_cathay.enrollments`<!-- .element: class="fragment" data-fragment-index="2" -->
- `SET search_path TO tenant_cathay;` <!-- .element: class="fragment" data-fragment-index="3" -->
- `SHOW search_path; 👉 "$user", public` <!-- .element: class="fragment" data-fragment-index="4" -->

</small>



</div>

<div class="right" style="font-size: 50%;">

`public.users` <!-- .element: class="fragment" data-fragment-index="5" -->
| | id  | organization_id |
| --- | --- | --- |
| | 42<!-- .element: class="fragment highlight-green" data-fragment-index="6" --> | 3<!-- .element: class="fragment highlight-green" data-fragment-index="6" --> |
| | 43<!-- .element: class="fragment highlight-green" data-fragment-index="6" --> | 3<!-- .element: class="fragment highlight-green" data-fragment-index="6" --> |
| | 44<!-- .element: class="fragment highlight-blue" data-fragment-index="6" --> | 4<!-- .element: class="fragment highlight-blue" data-fragment-index="6" --> |
<!-- .element: class="fragment" data-fragment-index="5" -->

`tenant_3.enrollments` <!-- .element: class="fragment" data-fragment-index="5" -->
| | id  | user_id | course_id |
| --- | --- | --- | --- |
| | 1110<!-- .element: class="fragment highlight-green" data-fragment-index="6" --> | 42<!-- .element: class="fragment highlight-green" data-fragment-index="6" --> | 90 |
| | 1112<!-- .element: class="fragment highlight-green" data-fragment-index="6" --> | 43<!-- .element: class="fragment highlight-green" data-fragment-index="6" --> | 101 |
<!-- .element: class="fragment" data-fragment-index="5" -->

`tenant_4.enrollments` <!-- .element: class="fragment" data-fragment-index="5" -->
| | id  | user_id | course_id |
| --- | --- | --- | --- |
| | 1111<!-- .element: class="fragment highlight-blue" data-fragment-index="6" --> | 44<!-- .element: class="fragment highlight-blue" data-fragment-index="6" --> | 84 |
<!-- .element: class="fragment" data-fragment-index="5" -->

</div>

Note:
- "Qualify name"
- Postgres Schema usage:
  - To allow many users to use one database without interfering with each other.
  - To organize database objects into logical groups to make them more manageable.
  - Third-party applications can be put into separate schemas so they do not collide with the names of other objects.
- `name` also moved to `tenant_x.user_profiles`


---

We use [apartment](https://github.com/rails-on-services/apartment) for switching tenant per request accroding to subdomain

<div> <!-- .element: class="fragment" -->

<small>

- "Current Tenant" is a global context
- the "Elevator" middleware

</small>

```ruby [1|5-6,13|3-7,13-14|8-11]
# GET https://cathay.business.hahow.in/foo/bar

# 🛢 SHOW search_path; 👉 "$user", public

# 1. the "Elevator" middleware
Tenant.switch!('cathay') { # 🛢 SET search_path TO tenant_cathay;
  # 🛢 SHOW search_path; 👉 tenant_cathay

  # 2. application logic
  render User.where(activated: true)
             .page(1) # => [User42, User43]

}
# 🛢 SHOW search_path; 👉 "$user", public
```

</div>

---

Bug! **but no data leakage.**<!-- .element: class="fragment" -->
```ruby
# Forget to switch tenant (e.g. in async job)
render User.where(activated: true).page(1) # => []
```

---

### Data isolation (2/4) - Storage

<div class="fragment">

1. **S3 Buckets**

<br>

- max 100 buckets by default
- max 1000 buckets upon request ([ref](https://docs.aws.amazon.com/AmazonS3/latest/userguide/BucketRestrictions.html))

</div></br>

<div class="fragment">

2. **S3 path prefix** **✅** <!-- .element: class="fragment" -->

```sh
/tenant_42/uploads/foo/bar/bez.png
/tenant_43/uploads/f/o/o/b/a/r.mp4
```
</div>

Notes:
- caveats:
  - two tenant key

---

### Data isolation (3/4) - Cache

<div> <!-- .element: class="fragment" -->

Redis key prefix

```sh
# Tenant private data
learning_leader_board:tenant_42  ->  [1,2,3]
learning_leader_board:tenant_43  ->  [1,2,3]
```
```sh
# Public data accross tenants
platform:hot_search_keywords     ->  ['Foo','Bar']
```

</div>

---

### Data isolation (4/4) - Search Engine

| | |
| --- | --- |
| Algolia✅ | <div>required `filters` for each API key</div><!-- .element: class="fragment" data-fragment-index="1" --> |
| PostgreSQL Full Text Search | Postgres Schema <!-- .element: class="fragment" data-fragment-index="2" --> |
| ElasticSearch | shards per-tenant + custom routing + required routing <!-- .element: class="fragment" data-fragment-index="3" --> |


<small>

 [ElasticSearch Multitenancy With Routing - Bozho's tech blog](https://techblog.bozho.net/elasticsearch-multitenancy-with-routing/) <!-- .element: class="fragment" data-fragment-index="3" -->
</small>

---

<!-- Skip this page for now -->
<!-- .slide: data-visibility="hidden" -->

### Onboarding

<div class="right" style="font-size:60%">

- POST /free_trial <!-- .element: class="fragment" data-fragment-index="1" -->
- Invitation email <!-- .element: class="fragment" data-fragment-index="2" -->
- Provision: <!-- .element: class="fragment" data-fragment-index="3" -->
  - New Postgres schema
  - New `Organization`
  - A root `User`

![provisioning](https://user-images.githubusercontent.com/12410942/163664795-942147ef-da04-4f2b-81d5-e3d1d34d63c1.png) <!-- .element: class="fragment" data-fragment-index="3" -->

</div>
<div class="left">

![request demo form](https://user-images.githubusercontent.com/12410942/163664694-0e4286ee-e01f-45df-8512-2dc74ce01049.png) <!-- .element: class="fragment" data-fragment-index="1" -->

![invitation email](https://user-images.githubusercontent.com/12410942/163664769-be5a1fc9-bb1a-48a5-8e35-0575e9a373e3.png) <!-- .element: class="fragment" data-fragment-index="2" -->

</div>

---


## Conclusion

|  | <small>Past:<br>1DB</small> | <small>Present:<br>1DB, N-schemas</small> | <small>Future:<br>N-DB, M-schemas |
| --- | --- | --- | --- |
| Operation | low | mid | ? |
| Multitenancy | ✅ | ✅ | ? |
| Logical Isolation |  | ✅ | ? |
| Physical Isolation |  |  | ? |
| Scale out |  |  | ? |

---
---

<small>

## (Possible) Future: M databases, N schemas
</small>

<div class="flex" style="font-size:30%">

  <div class="flex flex-col bg-red-600 rounded-xl m-2">
    <code class="m-1"> 🌐 CMS service </code>
    <ul>
      <li> Extract from Application </li>
      <li> Cacheable content </li>
    </ul>
    <div class="flex flex-col bg-blue-600 rounded-xl m-2">
      <code class="m-1"> 🛢 cms_databas </code>
      <div class="flex flex-col bg-sky-600 rounded-xl m-2">
        <code class="m-1"> 📂 public. </code>
        <div class="flex flex-col bg-teal-600 rounded-xl m-2">
          <code class="m-1"> 🗂 .courses </code>

| id | title |
| --- | --- |
| 956349fb-7085-4956-a353-db73ab1c9a0e | 簡報力 |
| e7c0ef97-1306-4efc-9948-9e12ed0e7972 | 設計思考 |
- ...
        </div>
      </div>
    </div>
  </div>

<div class="flex flex-col bg-red-600 rounded-xl m-2">
  <code class="m-1"> 🌐 Application </code>

  <div class="flex">
    <div class="flex flex-col bg-blue-600 rounded-xl m-2">
      <code class="m-1"> 🛢 application_database </code>
      <div class="flex flex-col bg-sky-600 rounded-xl m-2">
        <code class="m-1"> 📂 public. </code>
        <div class="flex flex-col bg-teal-600 rounded-xl m-2">
          <code class="m-1"> 🗂 .organizations </code>

<span>

| id | tenant_db_identifier |
| --- | --- |
| 1 | tenant_db_a |
| 2 | tenant_db_a |
| 3 | tenant_db_b_3 |
| 4 | tenant_db_c |
| 5 | tenant_db_c |
| 6 | tenant_db_c |
- ...
</span>
        </div>
      </div>
    </div>
    <div class="flex flex-col bg-blue-600 rounded-xl m-2">
      <code class="m-1"> 🛢 tenant_db_a </code>
      <div class="flex flex-col bg-sky-600 rounded-xl m-2">
        <code class="m-1"> 📂 tenant_1. </code>
        <div class="flex flex-col bg-teal-600 rounded-xl m-2">
          <code class="m-1"> 🗂 .enrollments </code>
<span>

| id  | user_id | cms_course_id |
| --- | --- | --- |
| 1110 | 42 | 956349fb-7085-4956-a353-db73ab1c9a0e |
| 1112 | 43 | e7c0ef97-1306-4efc-9948-9e12ed0e7972 |
- ...
</span>
        </div>
      </div>
      <div class="flex flex-col bg-sky-600 rounded-xl m-2">
        <code class="m-1"> 📂 tenant_2. </code>
        <div class="flex flex-col bg-teal-600 rounded-xl m-2">
          <code class="m-1"> 🗂 .enrollments </code>
        </div>
      </div>
    </div>
    <div class="flex flex-col bg-blue-600 rounded-xl m-2">
      <code class="m-1"> 🛢 tenant_db_b </code>
- Database Isolation for Tenant#3
      <div class="flex flex-col bg-sky-600 rounded-xl m-2">
        <code class="m-1"> 📂 tenant_3. </code>
        <div class="flex flex-col bg-teal-600 rounded-xl m-2">
          <code class="m-1"> 🗂 .enrollments </code>
        </div>
      </div>
    </div>
    <div class="flex flex-col bg-blue-600 rounded-xl m-2">
      <code class="m-1"> 🛢 tenant_db_c </code>
      <div class="flex flex-col bg-sky-600 rounded-xl m-2">
        <code class="m-1"> 📂 tenant_4. </code>
        <div class="flex flex-col bg-teal-600 rounded-xl m-2">
          <code class="m-1"> 🗂 .enrollments </code>
        </div>
      </div>
      <div class="flex flex-col bg-sky-600 rounded-xl m-2">
        <code class="m-1"> 📂 tenant_5. </code>
        <div class="flex flex-col bg-teal-600 rounded-xl m-2">
          <code class="m-1"> 🗂 .enrollments </code>
        </div>
      </div>
      <div class="flex flex-col bg-sky-600 rounded-xl m-2">
        <code class="m-1"> 📂 tenant_6. </code>
        <div class="flex flex-col bg-teal-600 rounded-xl m-2">
          <code class="m-1"> 🗂 .enrollments </code>
        </div>
      </div>
    </div>
  </div>

<!-- - [ ] Extract non-tenant data to another database (e.g. CMS service)
- [ ] Custom logic for selecting & creating `tenant_db` -->
</div>

---


## Conclusion

|  | <small>Past:<br>1DB</small> | <small>Present:<br>1DB, N-schemas</small> | <small>Future:<br>N-DB, M-schemas |
| --- | --- | --- | --- |
| Operation | low | mid | high |
| Multitenancy | ✅ | ✅ | ✅ |
| Logical Isolation |  | ✅ | ✅ |
| Physical Isolation |  |  | ✅ possible |
| Scale out |  |  | ✅ |

---
---

## Comparison

- Slack:
  - a `Workspace` is a tenant
  - Shard DB, MQ, search engining by groups of tenants
  - <span> [How Slack built shared channels](https://slack.engineering/how-slack-built-shared-channels/): TL;DR a multi-tenancy exception </span> <!-- .element class="fragment" -->

---

<div class="left left-more">

- Shopify
  - multiple databases, **multiple datacenters**
- 91App: No sure. Perhaps similar?<!-- .element class="fragment" -->
<!-- - GitLab: [1 database](https://docs.gitlab.com/ee/development/scalability.html#multi-tenancy), not really a multitenancy app? -->
- Please share about your company!<!-- .element class="fragment" -->

</div>
<div class="right right-less">

[<img width="100%" alt="image" src="https://user-images.githubusercontent.com/12410942/163684797-115dd4df-1261-43ee-a49f-3ad3e8c799e6.png">](https://youtu.be/F-f0-k46WVk)

</div>

---
---

## Q & A

<small> Slide deck: https://choznerol.github.io/talks/2022-04-27-multitenancy / [`source.md`](https://github.com/choznerol/talks/blob/main/2022-04-27-multitenancy/source.md) </small>

Note:
Multitenancy is a broad topic that I don't understand very much either
Ever implementation strategy in different company is unique and worth sharing!

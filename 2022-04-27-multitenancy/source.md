
## B2B SaaS Multitenancy
### a Hahow for Business case study

2022/04/27 @ AppWorks School system design study group <!-- .element: class="r-fit-text" -->

<!-- TODO: 2022/?/? @ Hahow -->

<small> slide deck: https://choznerol.github.io/talks/2022-04-27-multitenancy </small>
<small> markdown source: https://github.com/choznerol/talks/blob/main/2022-04-27-multitenancy/source.md </small>

---
---

## Agenda

1. What is multitenancy?
2. Multitenancy of Hahow for Business
   - Past: 1 DB
   - Present: 1 DB, N schemas
   - (Possible) Future: M DB, N schema
   - Conclusion
3. Comparison
4. Q & A

<small> slide deck: https://choznerol.github.io/talks/2022-04-27-multitenancy </small>

---
---

## 1. What is multitenancy?

1 deployment to serve N isolated "tenant"
![IBM Technology - Multitenancy Explained](https://user-images.githubusercontent.com/12410942/163659804-db9a58a9-cc90-44e7-8794-8fd873760bca.png)

[IBM Technology - Multitenancy Explained](https://youtu.be/60ccSmOxpMw)

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

Public data accross tenants: `Course`
<!-- .element: class="fragment" -->

Private tenant data: `Enrollment`
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
| id  | **organization_id** | name |
| --- | --- | --- |
| 42 | 3 | Jack |
| 43 | 3 | John |
| 44 | 4 | Jane |
<!-- .element: class="fragment" data-fragment-index="2" -->

`enrollments` <!-- .element: class="fragment" data-fragment-index="2" -->
| id  | user_id | course_id |
| --- | --- | --- |
| 1110 | 42 | 90 |
| 1111 | 44 | 84 |
| 1112 | 43 | 101 |
<!-- .element: class="fragment" data-fragment-index="2" -->

</div>
</div>

---

```ruby [1-5|7-8|10-13|15-16]
# List activated users
User.where(
  organization_id: current_organization.id,
  activated: true
).all # OK

# Bug! Data Leakage
User.where(activated: true)

# It can become hard to debug soon ...

# List enrollments of activated users
Enrollment.where(user: User.of_current_organization.activated) # OK

# Bug! Data Leakage!
Enrollment.where(user: User.activated)
```

---


## Conclusion

|  | 1DB | 1DB<br>N schemas | N DB<br>M schemas |
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

*<small>[PostgreSQL Schema](https://www.postgresql.org/docs/current/ddl-schemas.html) is like "namespace" for tables</small>* <!-- .element: class="fragment" data-fragment-index="1" -->

</div>

<div class="right" style="font-size: 50%;">

`public.users` <!-- .element: class="fragment" data-fragment-index="2" -->
| id  | organization_id |
| --- | --- |
| 42 | 3 |
| 43 | 3 |
| 44 | 4 |
<!-- .element: class="fragment" data-fragment-index="2" -->

`tenant_3.enrollments` <!-- .element: class="fragment" data-fragment-index="2" -->
| id  | user_id | course_id |
| --- | --- | --- |
| 1110 | 42 | 90 |
| 1112 | 43 | 101 |
<!-- .element: class="fragment" data-fragment-index="2" -->

`tenant_4.enrollments` <!-- .element: class="fragment" data-fragment-index="2" -->
| id  | user_id | course_id |
| --- | --- | --- |
| 1111 | 44 | 84 |
<!-- .element: class="fragment" data-fragment-index="2" -->

</div>

Note:
- `name` also moved to `tenant_x.user_profiles`


---

We use [apartment](https://github.com/rails-on-services/apartment) for switching tenant per request accroding to subdomain

<div> <!-- .element: class="fragment" -->

"Switching tenant" as a global context
```ruby
# GET https://foo-inc.business.hahow.in/bar

# In middleware
Tenant.switch!(Organization.of_subdomain(request.subdomain)) do

  render User.where(activated: true).page(1) # => [User42, User43]

end
```

</div>

<div> <!-- .element: class="fragment" -->

Bug! but no data leakage.
```ruby
# Forget to switch tenant (e.g. in async job)
render User.where(activated: true).page(1) # => []
```

</div>

---

### Data isolation (2/4) - Storage

<div>  <!-- .element: class="fragment" -->

1. **S3 Buckets**
- max 100 buckets by default
- max 1000 buckets upon request ([ref](https://docs.aws.amazon.com/AmazonS3/latest/userguide/BucketRestrictions.html))

</div></br>

<div>  <!-- .element: class="fragment" -->

2. **S3 path prefix** **✅** <!-- .element: class="fragment" -->

```
/tenant_42/uploads/foo/bar/bez.png
/tenant_43/uploads/f/o/o/b/a/r.mp4
```
</div>

---

### Data isolation (3/4) - Cache

<div> <!-- .element: class="fragment" -->

Redis key prefix

```
learning_leader_board:tenant_42  ->  [1,2,3]
learning_leader_board:tenant_43  ->  [1,2,3]
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

|  | 1DB | 1DB<br>N schemas | N DB<br>M schemas |
| --- | --- | --- | --- |
| Operation | low | mid | ? |
| Multitenancy | ✅ | ✅ | ? |
| Logical Isolation |  | ✅ | ? |
| Physical Isolation |  |  | ? |
| Scale out |  |  | ? |

---
---

<small>

## [WIP] (Possible) Future: M databases, N schemas

*TODO: Make a diagram instead ?* </small>

<div class="left left-more" style="font-size:30%">

- 🌐 CMS service # Cacheable content
  - 🛢`cms_database`
    - 📂`public.`
      - 🗂`courses`
- 🌐 Application
  - 🛢`application_database`
    - 📂`public.`
      - 🗂`organizations(id, tenant_db_identifier)`
  - 🛢`tenant_db_a`
    - 📂`tenant_1.`, 📂`tenant_2.`, 📂`tenant_3.`
      - 🗂`enrollments`, 🗂`enrollments`, 🗂`enrollments`
  - 🛢`tenant_db_exclusive_tenant_4` # Hard multitenancy for specific customer
    - 📂`tenant_4.`
      - 🗂`enrollments`
  - 🛢`tenant_db_b`
    - 📂`tenant_5.`, 📂`tenant_6.`, 📂`tenant_7.` ...
      - 🗂`enrollments`, 🗂`enrollments`, 🗂`enrollments`
  - ...

- [ ] Extract non-tenant data to another database (e.g. CMS service)
- [ ] Custom logic for selecting & creating `tenant_db`

</div>
<div class="right right-less" style="font-size:30%;">

**`cms_database/public`**`.courses`
| id | title |
| --- | --- |
| 956349fb-7085-4956-a353-db73ab1c9a0e | Foo |
| e7c0ef97-1306-4efc-9948-9e12ed0e7972 | Bar |

**`application_database/public`**`.organizations`
| id  | tenant_db_identifier
| --- | --- |
| 3 | `tenant_db_a` |
| 4 | `tenant_db_exclusive_tenant_4` |
| 5 | `tenant_db_b` |

**`tenant_db_a/tenant_3`**`.enrollments`
| id  | user_id | cms_course_id |
| --- | --- | --- |
| 1110 | 42 | 956349fb-7085-4956-a353-db73ab1c9a0e |
| 1112 | 43 | e7c0ef97-1306-4efc-9948-9e12ed0e7972 |

**`tenant_db_b/tenant_5`**`.enrollments`
| id  | user_id | cms_course_id |
| --- | --- | --- |
| 1111 | 44 | 956349fb-7085-4956-a353-db73ab1c9a0e |

</div>

---


## Conclusion

|  | 1DB | 1DB<br>N schemas | N DB<br>M schemas |
| --- | --- | --- | --- |
| Operation | low | mid | high |
| Multitenancy | ✅ | ✅ | ✅ |
| Logical Isolation |  | ✅ | ✅ |
| Physical Isolation |  |  | ✅ possible |
| Scale out |  |  | ✅ |

---
---

## Comparison

<div class="left left-more">

- Shopify: multiple databases, **multiple datacenters**

   <br>
<!-- - GitLab: [1 database](https://docs.gitlab.com/ee/development/scalability.html#multi-tenancy), not really a multitenancy app? -->
- 91App: No sure. Perhaps similar?
- **Please share about your company!**

</div>
<div class="right right-less">

[<img width="100%" alt="image" src="https://user-images.githubusercontent.com/12410942/163684797-115dd4df-1261-43ee-a49f-3ad3e8c799e6.png">](https://youtu.be/F-f0-k46WVk)

</div>

---
---

## Q & A

<small> slide deck: https://choznerol.github.io/talks/2022-04-27-multitenancy </small>
<small> markdown source: https://github.com/choznerol/talks/blob/main/2022-04-27-multitenancy/source.md </small>

Note:
Multitenancy is a broad topic that I don't understand very much either
Ever implementation strategy in different company is unique and worth sharing!

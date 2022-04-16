
## B2B SaaS Multitenancy
### a Hahow for Business case study

<div class="r-fit-text">
2022/04/27 @ AppWorks School system design study group </div>

<!-- TODO: 2022/?/? @ Hahow -->

<small> Slide: https://choznerol.github.io/talks/2022-04-27-multitenancy </small>
<small> Source: https://github.com/choznerol/talks/blob/main/2022-04-27-multitenancy/source.md </small>

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

<small> Slide: https://choznerol.github.io/talks/2022-04-27-multitenancy </small>

---
---

## 1. What is multitenancy?

1 deployment to serve N isolated "tenant"
![IBM Technology - Multitenancy Explained](https://user-images.githubusercontent.com/12410942/163659804-db9a58a9-cc90-44e7-8794-8fd873760bca.png)

[IBM Technology - Multitenancy Explained](https://youtu.be/60ccSmOxpMw)

---

## Objectives of multitenancy

- Covered in this talk:
  - Data isolation: Database, Storage, Cache
- Beyond this talk:
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

`Organization` has many `User`s

`User` has many `Enrollment`s

`Enrollment` belongs to `Course` and `User`

Public data accross tenants: `Course`

Private tenant data: `Enrollment`

---
---

## 2-1. Past: single database

<div class="left">

- 🌐 Application
  - 🛢 Databases
    - 🗂 Tables

</div>
<div class="right" style="font-size: 50%">

`users`
| id  | **organization_id** | name |
| --- | --- | --- |
| 42 | 3 | Jack |
| 43 | 3 | John |
| 44 | 4 | Jane |

`enrollments`
| id  | user_id | course_id |
| --- | --- | --- |
| 1110 | 42 | 90 |
| 1111 | 44 | 84 |
| 1112 | 43 | 101 |

</div>

---

```ruby
# List activated users
User.where(
  organization_id: current_organization.id,
  activated: true
).all # OK

User.where(activated: true) # Bug! Data Leakage
```

It can become hard to debug soon

```ruby
# List enrollments of activated users
Enrollment.where(user: User.of_current_organization.activated) # OK
Enrollment.where(user: User.activated) # Bug! Data Leakage!
```


---
---

## 2-2. Present: 1 database, N schemas

---

### Data isolation (1/4) - Database

<div class="left">

- 🌐 Application
  - 🛢 Databases
    -  📂 **[Schemas](https://www.postgresql.org/docs/current/ddl-schemas.html)**
       - 🗂 Tables

*<small>PostgreSQL Schema is like "namespace" for tables</small>*

</div>

<div class="right" style="font-size: 50%;">

**`public`**`.users`
| id  | organization_id | ~name~ |
| --- | --- | --- |
| 42 | 3 | ~Jack~ |
| 43 | 3 | ~John~ |
| 44 | 4 | ~Jane~ |

**`tenant_3`**`.enrollments`
| id  | user_id | course_id |
| --- | --- | --- |
| 1110 | 42 | 90 |
| 1112 | 43 | 101 |

**`tenant_4`**`.enrollments`
| id  | user_id | course_id |
| --- | --- | --- |
| 1111 | 44 | 84 |


</div>

Note:
- `name` also moved to `tenant_x.user_profiles`

---

We use [apartment](https://github.com/rails-on-services/apartment) for switching tenant per request accroding to subdomain

"Switching tenant" as a global context
```ruby
# GET https://cathaybank.business.hahow.in/foo/bar
Tenant.switch!(Organization.of_subdomain(request.subdomain)) do # In middleware

  render User.where(activated: true).page(1) # => [User42, User43]

end
```

Bug! but no data leakage.
```ruby
# Forget to switch tenant (e.g. in async job)

render User.where(activated: true).page(1) # => []
```

---

### Data isolation (2/4) - Storage

1. ✅ **S3 path prefix**

```
/tenant_42/uploads/foo/bar/bez.png
/tenant_43/uploads/f/o/o/b/a/r.mp4
```
<br>

2. **S3 Buckets**
- max 100 buckets by default
- max 1000 buckets upon request ([ref](https://docs.aws.amazon.com/AmazonS3/latest/userguide/BucketRestrictions.html))

---

### Data isolation (3/4) - Cache

Redis key prefix

```
learning_leader_board:tenant_42  ->  [1,2,3]
learning_leader_board:tenant_43  ->  [1,2,3]
platform:hot_search_keywords     ->  ['Foo','Bar']
```

---

### Data isolation (4/4) - Search Engine

| | |
| --- | --- |
| Algolia | required `filters` for each API key |
| PostgreSQL Full Text Search | Postgres Schema |
| ElasticSearch | Custom routing + per-tenant sharding|


<small>

 [ElasticSearch Multitenancy With Routing - Bozho's tech blog](https://paper.dropbox.com/ep/redirect/external-link?url=https%3A%2F%2Ftechblog.bozho.net%2Felasticsearch-multitenancy-with-routing%2F&hmac=mQNx1UZSgL%2BLcespdpwLVjBAlYImB%2BV9zNiyDnDE1%2BA%3D)
</small>

---

### Onboarding

<div class="right" style="font-size:60%">

- POST /free_trial
- custom check
- Invitation email
- Provision
  - New Postgres schema
  - free trial order
  - root user

![provisioning](https://user-images.githubusercontent.com/12410942/163664795-942147ef-da04-4f2b-81d5-e3d1d34d63c1.png)

</div>
<div class="left">

![request demo form](https://user-images.githubusercontent.com/12410942/163664694-0e4286ee-e01f-45df-8512-2dc74ce01049.png)

![invitation email](https://user-images.githubusercontent.com/12410942/163664769-be5a1fc9-bb1a-48a5-8e35-0575e9a373e3.png)

</div>

---
---

<small>

## (Possible) Future: M DBs, N schemas

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
- GitLab: [1 database](https://docs.gitlab.com/ee/development/scalability.html#multi-tenancy), not really a multitenancy app?
- 91App: No sure. Perhaps similar?
- **Please share about your company!**

</div>
<div class="right right-less">

[<img width="100%" alt="image" src="https://user-images.githubusercontent.com/12410942/163684797-115dd4df-1261-43ee-a49f-3ad3e8c799e6.png">](https://youtu.be/F-f0-k46WVk)

</div>

---
---

## Q & A

Note:
Multitenancy is a broad topic that I don't understand very much either
Ever implementation strategy in different company is unique and worth sharing!

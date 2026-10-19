# Student Project API — REST API müqaviləsi

> **Status:** Bu repoda hələ işlək server yoxdur. Bu sənəd və `openapi.yaml` API-nin razılaşdırılmış müqaviləsidir.
>
> Bütün `http://localhost:3000` ünvanları yalnız **planlaşdırılmış nümunədir**. Server implementasiya edilmədiyi üçün `curl` sorğuları hazırda uğurla işləməyəcək.

## Məqsəd

Student Project API tələbə layihələrini yaratmaq, siyahılamaq, tək layihəni oxumaq və qismən yeniləmək üçün nəzərdə tutulmuş sadə REST API-dir. Müqavilə **OpenAPI 3.1.0** əsasında təsvir edilir; maşın-oxunaqlı əsas mənbə `openapi.yaml` faylıdır.

**Baza URL (planlaşdırılmış):** `http://localhost:3000`

## Endpoint-lər

| Metod | Yol | Təsvir | Cavab statusları |
|---|---|---|---|
| `GET` | `/projects` | Bütün layihələri alır; nəticə `id` artan sırası ilə düz JSON array-dir. Body və məcburi parametr yoxdur. Boş kolleksiya da `200` və `[]` qaytarır. | `200 OK` |
| `GET` | `/projects/{id}` | Müsbət tam ədəd olan `id` üzrə bir layihəni alır. | `200 OK`, `404 Not Found`, `422 Unprocessable Content` |
| `POST` | `/projects` | Yeni layihə yaradır. `Content-Type: application/json` və JSON body məcburidir. | `201 Created`, `400 Bad Request`, `422 Unprocessable Content` |
| `PATCH` | `/projects/{id}` | Mövcud layihənin bir və ya bir neçə sahəsini qismən yeniləyir. `Content-Type: application/json` və JSON body məcburidir. | `200 OK`, `400 Bad Request`, `404 Not Found`, `422 Unprocessable Content` |

Müqavilə çərçivəsinə **authentication**, pagination, filter-lər, `PUT` və `DELETE` daxil deyil.

## `Project` modeli

### Uğurlu cavabdakı obyekt

Hər `Project` response obyektində aşağıdakı **6 sahənin hamısı məcburidir**; əlavə sahə olmur.

| Sahə | Tip və məhdudiyyət | Mənbə / izah |
|---|---|---|
| `id` | integer, minimum `1` | Server yaradır. |
| `name` | string, 1–100 simvol, ən azı bir qeyri-boşluq simvolu (`\S`) | `trim` və normalization tətbiq edilmir; məsələn, yalnız boşluqlardan ibarət ad etibarsızdır. |
| `description` | string, 0–1000 simvol | `POST` üçün default `""`-dir. |
| `status` | enum: `planned`, `in_progress`, `completed` | `POST` üçün default `planned`-dir. |
| `createdAt` | ISO 8601 UTC `date-time` | Server yaradır və sonradan dəyişmir. |
| `updatedAt` | ISO 8601 UTC `date-time` | Server yaradır; yalnız faktiki dəyər dəyişəndə yenilənir. |

### `POST /projects` body qaydaları

| Sahə | Qayda |
|---|---|
| `name` | **Məcburidir**; yuxarıdakı `name` məhdudiyyətlərinə cavab verməlidir. |
| `description` | Opsionaldır; verildikdə string və 0–1000 simvol olmalıdır. Verilməzsə `""`. |
| `status` | Opsionaldır; verildikdə yalnız `planned`, `in_progress`, `completed`. Verilməzsə `planned`. |
| `id`, `createdAt`, `updatedAt` | Client göndərə bilməz; göndərilərsə `422` qaytarılır. |
| Naməlum sahələr | Rədd edilir (`422`). |
| `null` | Bütün sahələr üçün rədd edilir (`422`). |

### `PATCH /projects/{id}` body qaydaları

- `name`, `description`, `status` sahələrinin heç biri təkbaşına məcburi deyil, amma body-də **ən azı biri** olmalıdır. Boş `{}` body-si `422`-dir.
- Göndərilməyən sahələr olduğu kimi qalır.
- `description: ""` təsviri qəsdən təmizləyir.
- Verilən hər sahə `POST`-dakı eyni tip, uzunluq, pattern və enum qaydalarına cavab verməlidir; `null`, server-idarəli və naməlum sahələr `422`-dir.
- Yeniləmə **atomikdir**: bir validation xətası varsa, heç bir sahə dəyişmir.
- Eyni dəyərlərlə PATCH yenə `200` qaytarır, lakin `updatedAt` (və əlbəttə `createdAt`) dəyişməz qalır.

## ID, body və validation qərarları

- Path `id` yalnız minimum `1` olan müsbət integer-dir. `abc`, `0`, mənfi dəyər və başqa malformed formatlar `422` verir.
- `GET /projects/{id}` üçün düzgün formatlı, lakin mövcud olmayan `id` `404` verir.
- `PATCH` üçün əvvəlcə **ID formatı və body validation** yoxlanır, sonra obyektin mövcudluğu yoxlanır. Buna görə həm body yanlış, həm də `id` mövcud deyilsə nəticə `422`-dir; düzgün body ilə mövcud olmayan `id` isə `404`-dür.
- JSON sintaksisi pozulubsa `400` qaytarılır. JSON sintaksiscə düzgündür, amma schema/rules pozulubsa `422` qaytarılır.
- Boş sorğu gövdəsi `422`-dir.
- Bir validation xətası belə bütün sorğunu rədd edir. Implementasiya aşkar etdiyi **bütün** validation səhvlərini `errors` massivində toplamalıdır.

## Xəta cavabı forması

Hər xəta cavabında `code`, `message` və `errors` sahələri var. Məsələn, `422` cavabı:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Göndərilən məlumat yanlışdır.",
  "errors": [
    {
      "field": "body.name",
      "message": "name sahəsi məcburidir."
    }
  ]
}
```

`404` üçün `errors` boş massivdir; `400` və `422` üçün ən azı bir element daşıyır. `field` səhvin yerini göstərir: `body`, `body.name`, `body.status`, `path.id` və ya başqa səhv sahə.

| Hal | Status və sabit dəyərlər |
|---|---|
| Layihə yoxdur | `404`; `code: "NOT_FOUND"`, `message: "Layihə tapılmadı."`, `errors: []` |
| Validation xətası | `422`; `code: "VALIDATION_ERROR"`, `message: "Göndərilən məlumat yanlışdır."` |
| Sintaksiscə səhv JSON | `400`; `code: "BAD_REQUEST"`, `message: "Sorğu gövdəsi düzgün JSON deyil."`, `errors: [{"field":"body","message":"JSON sintaksisi yanlışdır."}]` |

## HTTP nümunələri

Aşağıdakı ssenarilər **bir-birindən müstəqildir**, tarixlər isə illustrativdir. Nümunələrin baza layihəsi budur:

```json
{
  "id": 1,
  "name": "Tələbə portalı",
  "description": "Tələbələr üçün layihə idarəetməsi.",
  "status": "planned",
  "createdAt": "2026-10-19T12:00:00Z",
  "updatedAt": "2026-10-19T12:00:00Z"
}
```

### 1. Layihələri siyahılamaq — `200 OK`

**Sorğu**

```http
GET /projects HTTP/1.1
Host: localhost:3000
Accept: application/json
```

**Gözlənilən cavab**

```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "id": 1,
    "name": "Tələbə portalı",
    "description": "Tələbələr üçün layihə idarəetməsi.",
    "status": "planned",
    "createdAt": "2026-10-19T12:00:00Z",
    "updatedAt": "2026-10-19T12:00:00Z"
  }
]
```

### 2. `id=1` layihəsini almaq — `200 OK`

**Sorğu**

```http
GET /projects/1 HTTP/1.1
Host: localhost:3000
Accept: application/json
```

**Gözlənilən cavab**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 1,
  "name": "Tələbə portalı",
  "description": "Tələbələr üçün layihə idarəetməsi.",
  "status": "planned",
  "createdAt": "2026-10-19T12:00:00Z",
  "updatedAt": "2026-10-19T12:00:00Z"
}
```

### 3. Yeni layihə yaratmaq — `201 Created`

**Sorğu**

```http
POST /projects HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Accept: application/json

{
  "name": "Mobil tətbiq"
}
```

**Gözlənilən cavab**

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /projects/2

{
  "id": 2,
  "name": "Mobil tətbiq",
  "description": "",
  "status": "planned",
  "createdAt": "2026-10-19T12:05:00Z",
  "updatedAt": "2026-10-19T12:05:00Z"
}
```

### 4. Statusu yeniləmək — `200 OK`

**Sorğu**

```http
PATCH /projects/1 HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Accept: application/json

{
  "status": "in_progress"
}
```

**Gözlənilən cavab**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 1,
  "name": "Tələbə portalı",
  "description": "Tələbələr üçün layihə idarəetməsi.",
  "status": "in_progress",
  "createdAt": "2026-10-19T12:00:00Z",
  "updatedAt": "2026-10-19T12:30:00Z"
}
```

### 5. Mövcud olmayan layihəni almaq — `404 Not Found`

**Sorğu**

```http
GET /projects/999 HTTP/1.1
Host: localhost:3000
Accept: application/json
```

**Gözlənilən cavab**

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "code": "NOT_FOUND",
  "message": "Layihə tapılmadı.",
  "errors": []
}
```

### 6. İki validation xətası olan yaratma sorğusu — `422 Unprocessable Content`

**Sorğu**

```http
POST /projects HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Accept: application/json

{
  "name": "",
  "status": "finished"
}
```

**Gözlənilən cavab**

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/json

{
  "code": "VALIDATION_ERROR",
  "message": "Göndərilən məlumat yanlışdır.",
  "errors": [
    {
      "field": "body.name",
      "message": "Ad 1–100 simvol olmalı və ən azı bir qeyri-boşluq simvolu saxlamalıdır."
    },
    {
      "field": "body.status",
      "message": "Status planned, in_progress və ya completed olmalıdır."
    }
  ]
}
```

### 7. Düzgün body ilə mövcud olmayan layihəni yeniləmək — `404 Not Found`

**Sorğu**

```http
PATCH /projects/999 HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Accept: application/json

{
  "status": "completed"
}
```

**Gözlənilən cavab**

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "code": "NOT_FOUND",
  "message": "Layihə tapılmadı.",
  "errors": []
}
```

### 8. Boş PATCH body-si — `422 Unprocessable Content`

**Sorğu**

```http
PATCH /projects/1 HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Accept: application/json

{}
```

**Gözlənilən cavab**

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/json

{
  "code": "VALIDATION_ERROR",
  "message": "Göndərilən məlumat yanlışdır.",
  "errors": [
    {
      "field": "body",
      "message": "Ən azı name, description və ya status sahələrindən biri göndərilməlidir."
    }
  ]
}
```

### 9. Yanlış formatlı `id` — `422 Unprocessable Content`

**Sorğu**

```http
GET /projects/abc HTTP/1.1
Host: localhost:3000
Accept: application/json
```

**Gözlənilən cavab**

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/json

{
  "code": "VALIDATION_ERROR",
  "message": "Göndərilən məlumat yanlışdır.",
  "errors": [
    {
      "field": "path.id",
      "message": "id minimum 1 olan müsbət tam ədəd olmalıdır."
    }
  ]
}
```

### 10. Boş siyahı — `200 OK`

**Sorğu**

```http
GET /projects HTTP/1.1
Host: localhost:3000
Accept: application/json
```

**Gözlənilən cavab**

```http
HTTP/1.1 200 OK
Content-Type: application/json

[]
```

### 11. Sintaksiscə pozulmuş JSON — `400 Bad Request`

**Sorğu**

```http
POST /projects HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Accept: application/json

{"name":"Tələbə portalı"
```

**Gözlənilən cavab**

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "code": "BAD_REQUEST",
  "message": "Sorğu gövdəsi düzgün JSON deyil.",
  "errors": [
    {
      "field": "body",
      "message": "JSON sintaksisi yanlışdır."
    }
  ]
}
```

## Müqavilənin yoxlanması

Bu addımlar **yalnız `openapi.yaml` müqaviləsini** yoxlayır; server işə salmır və runtime/API testləri deyil. Layihənin kökündə aşağıdakıları ardıcıl icra edin:

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install PyYAML==6.0.3 openapi-spec-validator==0.7.2
python -c "import yaml; yaml.safe_load(open('openapi.yaml', encoding='utf-8')); print('YAML: OK')"
openapi-spec-validator openapi.yaml
```

Son iki əmrdə xəta yoxdursa YAML parse olunur və OpenAPI sənədi validator tərəfindən qəbul edilir.

## VS Code-da clone və açılış

Repo məxfidir: müəllimin [etikhacker/student-project-api](https://github.com/etikhacker/student-project-api) reposuna baxması üçün GitHub-da **collaborator girişi** tələb oluna bilər. Müəllimi repository collaborator-u kimi əlavə edin və ya təşkilati giriş qaydalarına uyğun oxuma icazəsi verin.

GitHub CLI ilə giriş edildikdən sonra istifadəçi öz kompüterində aşağıdakı əmrləri icra edə bilər:

```bash
gh repo clone etikhacker/student-project-api
cd student-project-api
code .
```

`code .` əmri VS Code-un Command Line Interface-i PATH-ə əlavə olduqda repunu VS Code-da açır. Bu addımlar istifadəçinin öz kompüterində yerinə yetirilir.

## Təqdimat checklist-i

- [x] `README.md` bu müqaviləni izah edir.
- [x] `openapi.yaml` OpenAPI **3.1.0** formatındadır və dörd əməliyyatı təsvir edir.
- [x] Ən azı 5 tələb olunur; burada 11 tam HTTP sorğu/cavab nümunəsi var.
- [x] `GET /projects` düz array, `id` artan sırası və boş siyahı üçün `200` qaydasını təsvir edir.
- [x] `POST` üçün `name` məcburi, `description` və `status` default-ları və naməlum/server sahələrinin rəddi göstərilib.
- [x] `PATCH` üçün ən azı bir yenilənə bilən sahə, atomiklik və timestamp qaydaları göstərilib.
- [x] `400`, `404` və `422` xəta formatları və validation ardıcıllığı göstərilib.
- [x] Nümunələrdən `201` cavabında `Location: /projects/{id}` var.
- [x] `python` və `openapi-spec-validator` yoxlamaları xətasız tamamlanıb.
- [ ] Müəllim private repo üçün collaborator/oxuma icazəsinə malikdir.

## Git ilə commit və push

Dəyişiklikləri yoxladıqdan sonra əsas budağa göndərmək üçün:

```bash
git add README.md openapi.yaml
git commit -m "docs: define student project REST API contract"
git push -u origin main
```

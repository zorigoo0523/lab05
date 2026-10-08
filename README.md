# Лаб 5 — API систем тест (Postman + Newman)

- Оюутны нэр: Ө.Зоригбаатар
- Оюутны код: B242270078
- Хичээл: F.CSA313 — Программ хангамжийн чанарын баталгаа ба тест (2026)

```
$ node -v
v22.22.2
$ newman -v
6.2.2
```

## Репозиторийн бүтэц

| Файл | Тайлбар |
| --- | --- |
| `server.js` | Тестлэх API (багшийн өгсөн, өөрчлөөгүй) |
| `lab05-collection.json` | Үндсэн collection — 21 бие даасан тест, `baseUrl` нь collection variable |
| `lab05-collection-fail.json` | Үндсэн collection-ий БҮТЭН хуулбар, зөвхөн T01-ийн НЭГ oracle (201 → 200) зориуд буруу |
| `lab05-collection-bugs.json` | Нэмэлт: `server.js`-ийн согогийг илрүүлэх тест (зөв загвараар шалгана, одоогийн сервер дээр унана) |
| `results/newman-pass.txt`, `newman-fail.txt`, `newman-down.txt` | Newman-ы бүтэн гаралт |
| `results/newman-bugs.txt`, `results/defects-curl.txt` | Согогийн нотолгоо |

Ажиллуулах:

```bash
node server.js                     # терминал 1
newman run lab05-collection.json   # терминал 2 (baseUrl collection дотроо байгаа тул --env-var хэрэггүй)
```

## Даалгавар 2 — Тест дизайн

### Сонголт ба төлөөлөх утгын хүснэгт

| Сонголт (функцийн оролт/төлөв) | Эквивалент анги | Төлөөлөх утга |
| --- | --- | --- |
| studentID-ийн төлөв | Идэвхтэй (`status` = active) | `PUT /students/t01-stu {"status":"active"}` |
| | Идэвхгүй (`status` ≠ active) | `"inactive"`, `"suspended"` |
| | Байхгүй (PUT хийгээгүй) | `t02-stu` (хэзээ ч үүсгээгүй) |
| Оюутны үзсэн хичээл ба урьдач нөхцөл | Урьдач бүгд үзсэн | taken `["CS201"]`, prereq `["CS201"]` |
| | Урьдач огт үзээгүй | taken `[]`, prereq `["CS201","CS202"]` |
| | Урьдач зарим нь үзсэн | taken `["CS201"]`, prereq `["CS201","CS301"]` |
| | Урьдач нөхцөлгүй хичээл | prereq `[]` (эсвэл орхисон) |
| courseID-ийн хүчинтэй байдал | Байгаа | `PUT /courses/t01-crs` |
| | Байхгүй | `t04-crs` (үүсгээгүй) |
| Хүсэлтийн формат | Хүчинтэй JSON, 2 талбар бүгд байгаа | `{"studentID":..,"courseID":..}` |
| | Талбар дутуу | зөвхөн studentID / зөвхөн courseID / `{}` |
| | Буруу JSON | `{"studentID": "x", "courseID": ` |

**Боломжгүй / утгагүй хослолууд:** оюутан байхгүй үед "урьдач хангана/хангахгүй" гэсэн сонголт утгагүй (харьцуулах `coursesTaken` байхгүй). Хичээл байхгүй үед "урьдач нөхцөлийн төлөв" мөн утгагүй. Иймд хослолыг 3×3×2×4 = 72 гэж бүгдийг нь биш, зөвхөн утга бүхий хослолоор хязгаарласан.

### Спецификацийн хүснэгт (21 тест = 21 Postman folder)

| № | Спецификаци | Төрөл | Хүлээгдэх статус | Хүлээгдэх result |
| --- | --- | --- | --- | --- |
| T01 | Идэвхтэй оюутан, урьдач хангана | Happy path | 201 | `OK` + registrationID (тоо) |
| T02 | Байхгүй оюутан | Алдааны анги | 200 | `ERROR_NO_STUDENT` |
| T03 | Идэвхгүй оюутан (`inactive`) | Алдааны анги | 200 | `ERROR_INACTIVE_STUDENT` |
| T04 | Байхгүй хичээл | Алдааны анги | 200 | `ERROR_NO_COURSE` |
| T05 | Урьдач огт хангаагүй | Алдааны анги | 200 | `ERROR_PREREQUISITES`, missing = `["CS201","CS202"]` |
| T06 | Урьдач хэсэгчлэн хангасан | Алдааны анги | 200 | `ERROR_PREREQUISITES`, missing = `["CS301"]` |
| T07 | Оюутан ч, хичээл ч байхгүй | Давхар алдаа | 200 | `ERROR_NO_STUDENT` |
| T08 | Идэвхгүй оюутан + байхгүй хичээл | Давхар алдаа | 200 | `ERROR_INACTIVE_STUDENT` |
| T09 | Идэвхгүй оюутан + урьдач дутуу | Давхар алдаа | 200 | `ERROR_INACTIVE_STUDENT` |
| T10 | Байхгүй оюутан + урьдачтай хичээл | Давхар алдаа | 200 | `ERROR_NO_STUDENT` |
| T11 | Урьдачгүй хичээл, хоосон coursesTaken | Хязгаар | 201 | `OK` |
| T12 | status/coursesTaken/prerequisites орхисон (анхдагч утга) | Хязгаар | 201 | `OK` |
| T13 | Шаардлагаас илүү хичээл үзсэн | Хязгаар | 201 | `OK` |
| T14 | status = `suspended` | Хязгаар | 200 | `ERROR_INACTIVE_STUDENT` |
| T15 | courseID дутуу | Хязгаар | 400 | `ERROR_BAD_REQUEST` |
| T16 | studentID дутуу | Хязгаар | 400 | `ERROR_BAD_REQUEST` |
| T17 | Хоосон объект `{}` | Хязгаар | 400 | `ERROR_BAD_REQUEST` |
| T18 | Буруу JSON | Хязгаар | 400 | `ERROR_BAD_JSON` |
| T19 | `GET /courses/:id` байгаа хичээл | Туслах функц | 200 | courseID, prerequisites зөв |
| T20 | `GET /courses/:id` байхгүй хичээл | Туслах функц | 404 | `ERROR_NO_COURSE` |
| T21 | Амжилттай бүртгэл `GET /registrations`-д орно | Туслах функц | 201 → 200 | registrationID, studentID, courseID таарна |

Тест бүр өөрийн ID-тай (`t01-stu`, `t01-crs` …) өөрийн PUT setup-тай тул бие даасан; registrationID-ийн яг утгыг шалгаагүй (зөвхөн `number` эсэх, T21-д POST-оос авсан утгаараа GET-ийг тулгасан).

**Нэг хүсэлтэд хэд хэдэн алдаа зэрэг гарвал** (T07–T10 ба T02/T03/T04-өөр тогтоосон): серверийн шалгах дараалал нь
`ERROR_BAD_REQUEST` → `ERROR_NO_STUDENT` → `ERROR_INACTIVE_STUDENT` → `ERROR_NO_COURSE` → `ERROR_PREREQUISITES`.

## Даалгавар 4 — Newman үр дүн

| Ажиллуулалт | Файл | requests | assertions executed | assertions failed | Exit code |
| --- | --- | --- | --- | --- | --- |
| PASS | `results/newman-pass.txt` | 47 | **75** | 0 | **0** |
| FAIL | `results/newman-fail.txt` | 47 | 75 | 1 | **1** |
| DOWN | `results/newman-down.txt` | 47 | 75 | 75 | **1** |

**Тестийн тоо: 21 тест (folder), 75 assertion** — `results/newman-pass.txt`-ийн `assertions executed` = 75.

- **PASS:** бүх 75 assertion амжилттай, exit code 0.
- **FAIL:** `lab05-collection-fail.json` нь үндсэн collection-ий бүтэн хуулбар бөгөөд зөвхөн T01-ийн статусын oracle-ийг 201 → 200 болгосон. Ирсэн статус 201 тул 1 assertion унаж exit code 1 болсон — CI quality gate-ийн зарчим: нэг oracle унавал pipeline зогсоно.
- **DOWN:** сервер унтарсан үед гаралтад `ECONNREFUSED` мөр гарна, exit code 1. Энэ бол **интерфейсийн алдаа** (серверт хүрч чадаагүй, хариу ирээгүй тул oracle-ийг үнэлж ч чадаагүй), харин FAIL бол **oracle-ийн алдаа** (хариу ирсэн ч хүлээсэн утгаас зөрсөн).

## Тестээр илэрсэн согогууд (`server.js`)

Нотолгоо: `results/newman-bugs.txt` (12 assertion, 6 унасан) ба `results/defects-curl.txt`.

| № | Согог | Бодит үйлдэл | Хүлээгдэх |
| --- | --- | --- | --- |
| D1 | `POST /registrations` биеийн утга `null` | Сервер **унана** (`Cannot destructure property 'studentID' of 'data' as it is null`) | 400 `ERROR_BAD_REQUEST` |
| D2 | courseID = `toString` (үүсгээгүй) | Сервер **унана** (`course.prerequisites` нь `undefined`) | 200 `ERROR_NO_COURSE` |
| D3 | studentID = `constructor` / `__proto__` (үүсгээгүй) | 200 `ERROR_INACTIVE_STUDENT` | 200 `ERROR_NO_STUDENT` |
| D4 | `GET /courses/constructor` (үүсгээгүй) | 200 `{"courseID":"constructor"}` | 404 |
| D5 | Нэг оюутныг нэг хичээлд давхар бүртгэх | 201 хоёулаа (registrationID 1, 2) | Давхар бүртгэлийг хориглох |

Шалтгаан: `students`, `courses` нь энгийн `{}` объект тул prototype-ийн түлхүүрүүд (`constructor`, `toString`, `__proto__`) "байгаа" мэт харагддаг. `Object.create(null)` эсвэл `Map` ашиглавал засагдана. D1-ийг `data = null` үед `const { studentID, courseID } = data` задлахаас үүдсэн.

## Дүгнэлт

Дизайны 5 алхмаас хамгийн их бодол шаардсан нь "сонголтууд → төлөөлөх утгууд" алхам байсан, учир нь аль нэг сонголт нөгөөгөөсөө хамаарч байгааг ялгах хэрэгтэй болсон. Боломжгүй хослол тохиолдсон: оюутан эсвэл хичээл байхгүй үед "урьдач хангана/хангахгүй" гэсэн сонголт утгагүй болдог тул 72 хослолыг бүгдийг нь биш, утга бүхий хослолоор хязгаарласан. Давхар алдааны тестүүд (T07–T10) серверийн шалгах дарааллыг нээсэн: NO_STUDENT нь INACTIVE-ээс, INACTIVE нь NO_COURSE ба PREREQUISITES-ээс түрүүлж шалгагддаг. Энэ дараалал зааварт бичигдээгүй тул тестээр тогтоох шаардлагатай байсан. Тест бүрийг өөрийн ID болон PUT setup-тай болгосноор сервер дахин асаалгүй давтан ажиллуулахад аюулгүй, бие даасан болсон. registrationID-ийн яг утгыг шалгахгүй байх нь тестийг ажиллуулах дарааллаас хамааралгүй болгосон. PASS, FAIL, DOWN гурван ажиллуулалт нь exit code-оор (0, 1, 1) ялгагддаг тул CI-д quality gate болгон ашиглаж болно. Тестүүд server.js-ээс 5 согог илрүүлсэн: `null` бие болон `toString` хичээл серверийг унагаадаг, `constructor`/`__proto__` түлхүүр буруу алдаа буцаадаг, мөн давхар бүртгэл хориглогддоггүй. Хамгийн ноцтой нь серверийг нэг хүсэлтээр унагаж чадах нь — бодит системд энэ нь үйлчилгээ тасалдуулах эмзэг байдал болно. Тест нь зөвхөн pass болгох биш, ийм согог олох зорилготой гэдгийг энэ лаборатори харуулсан.

> Даалгавар 5 (jsonplaceholder, нэмэлт) болон AI-ийн нэмэлт даалгаврыг хийгээгүй.

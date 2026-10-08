```markdown
# Лаборатори №5: API систем тест — Postman ба Newman

**Хичээл:** F.CSA313 — Программ хангамжийн чанарын баталгаа ба тест (2026)
**Оюутан:** А.Төгөлдөр
**Код:** B242270001

## Орчин

```bash
$ node -v
v22.22.1
$ newman -v
6.2.2

```

Хэрэгсэл: Postman (collection бүтээх), Newman 6.2.2 (CLI-аас ажиллуулах), Node.js (локал API `server.js`).

## Репозиторийн бүтэц

| Файл | Тайлбар |
| --- | --- |
| `server.js` | Тестлэх хичээлд бүртгүүлэх API (бэлэн өгөгдсөн) |
| `lab05-collection.json` | Үндсэн collection — бүх тест PASS (`baseUrl` хувьсагч агуулсан) |
| `lab05-collection-fail.json` | Зориуд нэг oracle-ийг буруу болгосон collection |
| `results/newman-pass.txt` | PASS ажиллуулалтын бүтэн гаралт |
| `results/newman-fail.txt` | FAIL ажиллуулалтын бүтэн гаралт |
| `results/newman-down.txt` | DOWN (сервер унтарсан) ажиллуулалтын бүтэн гаралт |

## Ажиллуулах

```bash
node server.js                      # терминал 1
mkdir -p results                    # терминал 2

# Collection дотор baseUrl variable хадгалагдсан тул шууд ажиллана:
newman run lab05-collection.json 2>&1 | tee results/newman-pass.txt
newman run lab05-collection-fail.json 2>&1 | tee results/newman-fail.txt

# node server.js-ээ унтраасны (Ctrl+C) дараа:
newman run lab05-collection.json 2>&1 | tee results/newman-down.txt

```

---

## Даалгавар 2: Тест дизайн

### Сонголтууд ба төлөөлөх утгууд

`POST /registrations` функцийн оролт: `studentID`, `courseID`, мөн сервер дээр урьдчилан тавигдсан оюутан ба хичээлийн төлөв.

| № | Сонголт | Эквивалент анги | Төлөөлөх утга |
| --- | --- | --- | --- |
| 1 | studentID-гийн хүчинтэй байдал | Идэвхтэй оюутан (`status="active"`) | `S1`, `S5`, `S8`, `S9` |
|  |  | Идэвхгүй оюутан (`status≠"active"`) | `S3`, `S6` (`inactive`) |
|  |  | Байхгүй оюутан | `S_NOT_FOUND` |
| 2 | Оюутны үзсэн хичээлүүд | Урьдач нөхцөлийг бүрэн хангана | `coursesTaken=["CS201"]` (S1) |
|  |  | Урьдач нөхцөлийг хэсэгчлэн хангана | `coursesTaken=["CS101"]` (S9) |
|  |  | Урьдач нөхцөлийг огт хангахгүй | `coursesTaken=[]` (S5) |
| 3 | courseID-гийн хүчинтэй байдал | Хичээл байгаа | `C1`, `C2`, `C3`, `C5`, `C8`, `C9` |
|  |  | Хичээл байхгүй | `C_NOT_FOUND`, `C_NONE` |
| 4 | Хичээлийн урьдач нөхцөл | Урьдач нөхцөлгүй (`[]`) | `C8`: `prerequisites=[]` |
|  |  | Бүгдийг нь үзсэн байхыг шаардана | `C1`: `["CS201"]` |
|  |  | Олон хичээл шаардана (зарим нь дутуу) | `C9`: `["CS101", "CS102"]` |
|  |  | Огт үзээгүй байхыг шаардана | `C5`: `["CS101"]` |
| 5 | Хүсэлтийн формат | Хоёр талбар бүгд байна | `{"studentID","courseID"}` |
|  |  | `courseID` дутуу | `{"studentID":"S7"}` |

**Боломжгүй хослол:** байхгүй оюутанд `status` байхгүй тул «байхгүй оюутан + идэвхгүй» гэсэн хослол байж болохгүй. Мөн байхгүй оюутан эсвэл байхгүй хичээлийн хувьд урьдач нөхцөлийг үнэлэх боломжгүй. Тиймээс эдгээр хослолыг тусад нь тест болгоогүй.

### Спецификацийн хүснэгт

Серверийн шалгах дараалал: `BAD_REQUEST → NO_STUDENT → INACTIVE_STUDENT → NO_COURSE → PREREQUISITES → OK`.

| № | Спецификац (collection дахь нэр) | Төрөл | Оролт (studentID / courseID) | Setup | Хүлээгдэх статус | Хүлээгдэх result |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Happy Path | Happy path | S1 / C1 | S1: active, `["CS201"]`; C1: `["CS201"]` | 201 | `OK` + `registrationID` (тоо) |
| 2 | ERROR_NO_STUDENT | Алдааны анги | S_NOT_FOUND / C2 | C2: `[]` | 200 | `ERROR_NO_STUDENT` |
| 3 | ERROR_INACTIVE_STUDENT | Алдааны анги | S3 / C3 | S3: inactive; C3: `[]` | 200 | `ERROR_INACTIVE_STUDENT` |
| 4 | ERROR_NO_COURSE | Алдааны анги | S4 / C_NOT_FOUND | S4: active, `[]` | 200 | `ERROR_NO_COURSE` |
| 5 | ERROR_PREREQUISITES | Алдааны анги | S5 / C5 | S5: active, `[]`; C5: `["CS101"]` | 200 | `ERROR_PREREQUISITES` |
| 6 | Double Error (Inactive + No Course) | Давхар алдаа | S6 / C_NONE | S6: inactive | 200 | `ERROR_INACTIVE_STUDENT` |
| 7 | ERROR_BAD_REQUEST | Хязгаарын тохиолдол | S7 / (courseID дутуу) | — | 400 | `ERROR_BAD_REQUEST` |
| 8 | No Prerequisites Course | Хязгаарын тохиолдол | S8 / C8 | S8: active, `[]`; C8: `[]` | 201 | `OK` |
| 9 | ERROR_PREREQUISITES_PARTIAL | Хэсэгчилсэн урьдач | S9 / C9 | S9: active, `["CS101"]`; C9: `["CS101", "CS102"]` | 200 | `ERROR_PREREQUISITES` |

Спецификац бүр бие даасан: өөрийн оюутан/хичээлийг (S1…S9, C1…C9) өөрийн pre-request script эсвэл тусдаа хүсэлтээр `PUT`-аар тавина. `PUT` нь бичлэгийг бүхэлд нь дахин бичдэг тул давтан ажиллуулахад аюулгүй.

---

## Даалгавар 4: Newman-аар автоматжуулсан үр дүн

Бүх цуглуулга дээр `baseUrl` хувьсагч тохируулагдсан тул Newman-ийг нэмэлт параметргүйгээр шууд ажиллуулсан. Бүтэн гаралт нь `results/` хавтсанд байгаа.

| Ажиллуулалт | Файл | requests (executed / failed) | assertions (executed / failed) | Exit code |
| --- | --- | --- | --- | --- |
| PASS | `results/newman-pass.txt` | 22 / 0 | **19 / 0** | **0** |
| FAIL | `results/newman-fail.txt` | 22 / 0 | 19 / **1** | **1** |
| DOWN | `results/newman-down.txt` | 22 / 22 | 19 / **19** | **1** |

**Нийт тест:** 9 бие даасан тест кейс (9 `POST` + 13 setup `PUT` = 22 request), **19 assertion** (Happy Path 3, Double Error 2, Partial Prerequisites 2, бусад 6 тест тус бүр 2). Энэ тоо `results/newman-pass.txt`-ийн `assertions executed = 19`-тэй яг таарна.

### PASS

Бүх 19 assertion амжилттай, `failed = 0`, exit code **0**.

### FAIL

`lab05-collection-fail.json` нь үндсэн collection-оос зөвхөн нэг мөрөөр ялгаатай. Happy Path дахь `Status is 201` oracle нь 201-ийн оронд `200` хүлээнэ. Сервер 201 буцаасан тул `expected response to have status code 200 but got 201` алдаа гарч, `assertions failed = 1`, exit code **1** болсон. Энэ нь CI quality gate-ийн зарчим: нэг oracle унавал pipeline амжилтгүй болно.

### DOWN

Серверийг Ctrl+C-ээр унтрааж ажиллуулахад бүх 22 request `connect ECONNREFUSED 127.0.0.1:3000` алдаа өгч, exit code **1** болсон.

**Интерфейсийн алдаа ба oracle-ийн алдааны ялгаа:** DOWN бол интерфейсийн алдаа (сервертэй холбогдож чадаагүй тул хариу огт ирээгүй), харин FAIL бол oracle-ийн алдаа (сервер хариу өгсөн боловч хариу нь хүлээгдсэнээс өөр байсан). DOWN үед `assertions failed = 19` гарсан нь тест тус бүр буруу гэсэн үг биш, хариу байхгүй тул шалгах зүйл олдоогүйн үр дагавар юм.

---

## Дүгнэлт

Дизайны 5 алхмаас хамгийн их бодол шаардсан нь сонголтуудыг тодорхойлж, дараа нь тэдгээрийн хослолоос бодит тест кейс гаргах алхам байлаа. Боломжгүй хослол таарсан: байхгүй оюутан нь `status`-гүй тул «байхгүй + идэвхгүй» гэж байж болохгүй, байхгүй хичээлийн урьдач нөхцөлийг ч шалгах боломжгүй. Давхар алдааны тест (идэвхгүй оюутан + байхгүй хичээл) нь спецификацид заагаагүй шалгах дарааллыг тогтоосон бөгөөд сервер `ERROR_INACTIVE_STUDENT`-ийг түрүүлж буцаадаг нь батлагдсан. Тестүүд системд согог илрүүлээгүй, 9 тест бүгд хүлээгдсэн үр дүнтэй таарсан (19/19).

Өмнөх туршилтаар орхигдуулж байсан «зарим урьдач нөхцөл дутуу» тохиолдлыг (`ERROR_PREREQUISITES_PARTIAL`) шинээр нэмж шалгаснаар тестийн хамрах хүрээг амжилттай өргөжүүллээ. Түүнчлэн Postman collection дотор `baseUrl` хувьсагчийг хадгалснаар CLI орчинд Newman-ийг нэмэлт параметргүйгээр илүү хялбар ажиллуулах боломжтой болсон. FAIL болон DOWN ажиллуулалтаар oracle нь согогийг хэрхэн барьж чадаж байгааг болон интерфейсийн алдаанаас ямар ялгаатайг харж, Newman-ийн exit code (0/1) нь CI дээр автомат quality gate болох боломжтойг бүрэн ойлголоо.

```

```
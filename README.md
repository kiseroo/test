# Frontend ба Backend хэрхэн холбогддог вэ — Маш дэлгэрэнгүй тайлбар

> Программчлал огт мэдэхгүй хүнд зориулсан

---

## НЭГДҮГЭЭР ХЭСЭГ: Суурь ойлголтууд

---

### `export` гэж юу вэ?

JavaScript/TypeScript файл бүр нь **тусдаа хайрцаг** шиг.
Дотор байгаа зүйлийг гаднаас харах боломжгүй — `export` гэж бичсэн тохиолдолд л гаднаас ашиглаж болно.

```ts
// a.ts
const нэр = "Монх";            // export хийгүй → гадна харахгүй
export const нас = 25;         // export хийсэн → гадна харж болно
```

```ts
// b.ts
import { нас } from "./a";    // a.ts-аас нас-г авна
console.log(нас);              // 25
```

---

### `const` гэж юу вэ?

Хувьсагч (variable) зарлах арга. Утгыг нэг л удаа тогтооно, дараа өөрчлөгдөхгүй.

```ts
const нэр = "Монх";
нэр = "Бат";   // ❌ Алдаа! const өөрчлөгдөхгүй
```

---

### `async / await` гэж юу вэ?

Зарим үйлдлүүд **цаг хугацаа зарцуулдаг** — интернэтээс өгөгдөл татах, файл уншихх гэх мэт.
JavaScript нь хүлээхгүйгээр дараагийн мөрийг гүйцэтгэдэг. Энэ асуудлыг шийдэхийн тулд `async/await` ашиглана.

**`async` байхгүй бол (буруу):**
```ts
const loginUser = (data) => {
  const response = api.post("/login", data);
  // api.post цаг авна, харин JS хүлээхгүй
  // response нь өгөгдөл биш Promise объект байна → алдаа гарна
  return response.data;  // undefined байна!
};
```

**`async/await` ашигласан (зөв):**
```ts
const loginUser = async (data) => {
  const response = await api.post("/login", data);
  // await = "дуустал хүлээ"
  // response нь одоо бодит өгөгдөл байна
  return response.data;  // { id: 1, email: "..." }
};
```

`async` гэдэг нь "энэ функц доторх `await`-г ажиллуулна" гэсэн үг.
`await` гэдэг нь "энэ мөр дуусталхүлээ, дараа нь үргэлж" гэсэн үг.

---

### `interface` / `type` гэж юу вэ? (TypeScript)

TypeScript нь JavaScript дээр **төрлийн шалгалт** нэмсэн хэл.
`interface` нь объектийн "загвар" тодорхойлно — ямар property байх ёстойг заана.

```ts
interface LoginDto {
  email: string;     // email нь заавал байх, string байх ёстой
  password: string;  // password нь заавал байх, string байх ёстой
}
```

Энэ `LoginDto` нь `api.ts`-д байгаа бөгөөд Swagger-аас автоматаар үүссэн.
Backend-ийн `/v1/auth/login` endpoint нь яг энэ бүтэцтэй өгөгдөл хүлээж авдаг.

```ts
// ЗӨВ:
const data: LoginDto = { email: "монх@mail.com", password: "12345678" };

// БУРУУ (TypeScript алдаа гаргана):
const data: LoginDto = { email: "монх@mail.com" };  // password дутуу!
const data: LoginDto = { email: 123, password: "12345678" };  // email нь тоо биш байх ёстой
```

---

## ХОЁРДУГААР ХЭСЭГ: Файл тус бүрийн мөр бүрийн тайлбар

---

### `interceptor.ts` — Axios тохиргоо

```ts
import axios from "axios";
```
`axios` нь HTTP request хийдэг library (бэлэн хэрэгсэл).
`npm install axios` хийгдсэн байгаа тул `node_modules`-аас авна.

```ts
const api = axios.create({
```
`axios.create()` нь **тохируулга хийсэн axios instance** үүсгэнэ.
Энгийн `axios`-г шууд ашиглаж болох ч `axios.create()` ашигласнаар
тохиргоо (baseURL, withCredentials) бүх request-т автоматаар хэрэглэгдэнэ.

```ts
  baseURL: "http://localhost:3001",
```
`baseURL` = бүх request-ийн эхний хэсэг.

```
api.get("/v1/auth/me")
→ "http://localhost:3001" + "/v1/auth/me"
→ "http://localhost:3001/v1/auth/me"
```

`localhost` = өөрийн компьютер (production-д жинхэнэ domain болно)
`3001` = backend-ийн port дугаар (NestJS тэр port дээр ажиллаж байна)

```ts
  withCredentials: true,
```
`withCredentials: true` = "cookie илгээхийг зөвшөөр".

Frontend `localhost:5173`, Backend `localhost:3001` — **өөр origin**.
Browser нь аюулгүй байдлын дүрмээр өөр origin руу cookie илгээхгүй.
`withCredentials: true` тэр хоригийг гаргана.

```ts
});
```
`axios.create({...})` дуусна. `api` хувьсагчид хадгална.

```ts
api.interceptors.response.use(
```
`interceptors` = "саатуулагч". Response ирэхэд **дундаас барьж** шалгана.
Хоолны захиалгын хувьд: захиалга өгч → захиалга ирэхэд зөв эсэхийг шалгана → авна.
Энд: request илгээж → response ирэхэд 401 эсэхийг шалгана → component-д өгнө.

```ts
  (response) => response,
```
Алдаагүй response ирвэл (200, 201...) шууд буцаана. Юу ч хийхгүй.

```ts
  (error) => {
```
Алдааны response ирвэл (400, 401, 500...) энэ функц ажиллана.

```ts
    if (error.response?.status === 401 && window.location.pathname !== "/login") {
```
`error.response?.status` = HTTP status code (жишээ нь `401`).
`?.` = "байгаа бол авна, байхгүй бол undefined" (optional chaining).
`window.location.pathname` = одоо байгаа URL-ийн зам (`"/login"`, `"/"` гэх мэт).

Нөхцөл: `401 ирсэн` **БА** `одоо /login дээр биш`.

```ts
      window.location.href = "/login";
```
Browser-г `/login` хуудас руу шилжүүлнэ. (Хуучин URL-г орхино)

```ts
    }
    return Promise.reject(error);
```
Алдааг үргэлжлүүлэн дамжуулна — `useMutation`-ийн `onError` эсвэл `try/catch` барина.

```ts
export default api;
```
Бусад файлд ашиглах боломжтой болгоно. `default` = нэргүй import хийж болно:
```ts
import axiosInstance from "@/libs/interceptor";  // хүссэн нэрийг өгнө
```

---

### `auth.service.ts` — Api instance ба функцүүд

```ts
import type { LoginDto, RegisterDto } from "@/libs/api";
```
`import type` = зөвхөн TypeScript төрлийг авна. JavaScript код биш.
`LoginDto`, `RegisterDto` = `api.ts`-д байгаа interface-үүд.
`@/libs/api` = `src/libs/api.ts` файл (`@` нь `src/` гэсэн alias).

```ts
import { Api } from "@/libs/api";
```
`api.ts`-аас `Api` class авна. Энэ class нь backend-ийн бүх endpoint-ийн
TypeScript wrapper агуулж байна (Swagger-аас автоматаар үүссэн).

```ts
import axiosInstance from "@/libs/interceptor";
```
`interceptor.ts`-аас `export default api` буюу тохируулга хийсэн axios instance авна.

```ts
export const api = new Api();
```
`new Api()` = Api class-аас **объект үүсгэнэ** (instantiation).
`new` гэдэг нь "ене загвараар шинэ объект үүсгэ" гэсэн үг.

Дотроо юу болдог вэ?
```ts
// api.ts дахь Api class (хялбарчилсан):
class Api {
  instance: AxiosInstance;  // axios instance хадгалах газар

  constructor() {
    this.instance = axios.create({ baseURL: "" });  // шинэ axios үүсгэнэ
  }

  // v1 дахь бүх endpoint функцүүд...
}
```
`new Api()` хийхэд `constructor` ажиллаж **шинэ хоосон axios** үүсгэнэ.

```ts
api.instance = axiosInstance;
```
Дотроо байгаа тэр хоосон axios-г **interceptor тохируулсан axios-аар солино**.

Яагаад ийм 2 алхам хийх вэ?
Учир нь `Api` class нь Swagger-аас **автоматаар үүссэн** тул бид засаж болохгүй.
Гэвч `instance` property нь `public` тул гаднаас солиж болно.
Тиймээс эхлээд `new Api()` хийж, дараа `api.instance = axiosInstance` гэж солино.

```ts
export const loginUser = async (data: LoginDto) => {
```
- `export` = гаднаас ашиглаж болно (providers.tsx import хийнэ)
- `const loginUser` = `loginUser` нэртэй хувьсагчид функц хадгалана
- `async` = энэ функц доторх `await` ажиллана
- `(data: LoginDto)` = нэг параметр авна, нэр нь `data`, төрөл нь `LoginDto`
  - Яагаад `LoginDto` вэ? Учир нь backend-ийн login endpoint `{ email, password }` хүлээнэ
  - TypeScript алдааг урьдчилан барьна: `data.email` байхгүй бол compile алдаа гарна

```ts
  const response = await api.v1.authControllerLoginV1(data);
```
- `api.v1` = `Api` class-ийн `v1` объект (v1 version-ийн endpoint-үүд)
- `authControllerLoginV1` = `POST /v1/auth/login` endpoint-ийн wrapper функц
- `(data)` = `{ email, password }` өгөгдлийг дамжуулна
- `await` = response ирэхийг хүлээнэ
- `response` = backend-ийн хариу: `{ data: {...}, status: 200, headers: {...} }`

`api.ts`-д энэ функц иймэрхүү харагдана:
```ts
authControllerLoginV1: (data: LoginDto) =>
  this.request({
    path: `/v1/auth/login`,
    method: "POST",
    body: data,
    type: "application/json",
  }),
```
Яг `POST http://localhost:3001/v1/auth/login` руу `data`-г JSON болгон илгээнэ.

```ts
  return response.data;
```
`response` нь Axios-ийн бүтэц:
```
{
  data: { ... },      ← backend-аас ирсэн бодит өгөгдөл
  status: 200,
  headers: { ... },
  config: { ... },
}
```
Бидэнд зөвхөн `data` хэсэг хэрэгтэй тул `response.data` буцаана.

---

## ГУРАВДУГААР ХЭСЭГ: Бүтэн flow — Хэрэглэгчийн нүднээс харах

---

### Хэрэглэгч login form бөглөж "Login" дарна

```
┌─────────────────────────────────────┐
│  Email:    монх@mail.com            │
│  Password: ••••••••                 │
│                                     │
│         [  Login  ]  ← дарлаа      │
└─────────────────────────────────────┘
```

### Алхам 1: LoginForm.tsx — товч дарагдав

```ts
// LoginForm.tsx (хялбарчилсан)
<form onSubmit={form.handleSubmit(onSubmit)}>
```
`form.handleSubmit(onSubmit)` = React Hook Form-ийн механизм.
1. Validation шалгана (email формат зөв үү, password 6+ тэмдэгт үү...)
2. Зөв бол `onSubmit(data)` дуудна, data = `{ email: "монх@mail.com", password: "12345678" }`

### Алхам 2: useLoginForm.ts — onSubmit дуудагдав

```ts
const onSubmit = async (data: LoginFormData) => {
  try {
    await login(data);   // providers.tsx-ийн login функц дуудна
    navigate("/");       // амжилттай бол "/" руу явна
  } catch {
    // алдаа гарвал энд барина (loginError state-д харагдана)
  }
};
```

`data` энд юу байна вэ?
```ts
data = {
  email: "монх@mail.com",
  password: "12345678",
  // confirmPassword гэж байхгүй — LoginFormData зөвхөн email, password
}
```

### Алхам 3: providers.tsx — loginMutation ажиллана

```ts
const login = async (data: LoginDto): Promise<void> => {
  await loginMutation.mutateAsync(data);
};
```

`loginMutation.mutateAsync(data)`:
- TanStack Query-д "login хийх үйлдэл эхэллэ" гэж мэдэгдэнэ
- `isPending = true` болно → button disabled болж, spinner харагдаж болно
- `mutationFn: loginUser` дуудагдана → `auth.service.ts`-ийн `loginUser(data)`

### Алхам 4: auth.service.ts — loginUser ажиллана

```ts
export const loginUser = async (data: LoginDto) => {
  const response = await api.v1.authControllerLoginV1(data);
  return response.data;
};
```

`api.v1.authControllerLoginV1(data)` дуудагдахад:
- `api.ts`-д байгаа функц ажиллана
- `api.instance` (= interceptor-той axios) ашиглана
- HTTP request үүснэ

### Алхам 5: interceptor.ts → Network — HTTP request явна

```
POST http://localhost:3001/v1/auth/login
Content-Type: application/json
Cookie: (хоосон, анх нэвтрэхийн өмнө cookie байхгүй)

{
  "email": "монх@mail.com",
  "password": "12345678"
}
```

Browser-ийн Network tab дээр энийг харж болно.

### Алхам 6: Backend (NestJS) — хүлээн авна

```
Backend юу хийх вэ:
1. /v1/auth/login route барина
2. email-ээр хэрэглэгч DB-д байгаа эсэх шалгана
3. password-г bcrypt ашиглан шалгана
4. Зөв бол session үүсгэж cookie-д хийнэ
5. Response буцаана
```

### Алхам 7: Response ирнэ

**Амжилттай бол (200 OK):**
```
HTTP/1.1 200 OK
Set-Cookie: session=xK9mP2...; HttpOnly; Path=/   ← backend cookie тавина

{
  "message": "Login successful"
}
```

`Set-Cookie` header = browser-д "энэ cookie-г хадгал" гэж зааварлана.
`HttpOnly` = JavaScript-ээр cookie уншиж болохгүй (аюулгүй байдал).
Browser тэр cookie-г хадгалж, дараагийн request бүрт автоматаар илгээнэ.

**Алдаатай бол (401):**
```
HTTP/1.1 401 Unauthorized

{
  "message": "Нууц үг буруу байна",
  "statusCode": 401
}
```

### Алхам 8: providers.tsx — onSuccess эсвэл onError

**Амжилттай бол:**
```ts
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ["currentUser"] });
}
```
`invalidateQueries` = "currentUser" кэшийг "хуучирсан" гэж тэмдэглэнэ.
TanStack Query `getMe()` функцийг дахин автоматаар дуудна.

### Алхам 9: getMe() дуудагдана

```ts
export const getMe = async () => {
  const response = await api.v1.authControllerMeV1();
  return response.data;
};
```

```
GET http://localhost:3001/v1/auth/me
Cookie: session=xK9mP2...   ← browser автоматаар нэмнэ (withCredentials: true учраас)
```

Backend cookie-г шалгаж, хэрэглэгчийн мэдээлэл буцаана:
```json
{ "id": 1, "name": "Монх", "email": "монх@mail.com" }
```

### Алхам 10: State шинэчлэгдэж, хуудас өөрчлөгдөнө

```ts
const { data: user } = useQuery({ queryKey: ["currentUser"], queryFn: getMe });
const isAuthenticated = !!user;
// user = { id:1, name:"Монх" } → !!user = true
```

`!!` = boolean болгоно. `!!{ id:1 }` = `true`. `!!undefined` = `false`.

```ts
// router.tsx
const ProtectedRoute = () => {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? <Outlet /> : <Navigate to="/login" />;
};
```

`isAuthenticated = true` болсон тул `<Outlet />` буцаана → HomePage харагдана.

`useLoginForm.ts`-ийн `navigate("/")` ажиллаж хуудас шилжинэ.

---

## ДӨРӨВДҮГЭЭР ХЭСЭГ: Мөр бүрийн хураангуй

```ts
// auth.service.ts

import type { LoginDto, RegisterDto } from "@/libs/api";
// └─ TypeScript-д зориулсан төрөл (interface) авна

import { Api } from "@/libs/api";
// └─ Swagger-аас үүссэн Api class авна (бүх endpoint функц агуулна)

import axiosInstance from "@/libs/interceptor";
// └─ baseURL, withCredentials, 401-redirect тохируулсан axios авна

export const api = new Api();
// └─ Api class-аас объект үүсгэнэ (дотроо хоосон axios instance үүснэ)

api.instance = axiosInstance;
// └─ Тэр хоосон axios-г interceptor-той axios-аар солино

export const loginUser = async (data: LoginDto) => {
// └─ гаднаас дуудаж болох, async, LoginDto бүтэцтэй параметр авна

  const response = await api.v1.authControllerLoginV1(data);
// └─ POST /v1/auth/login дуудна, дуусталхүлээнэ

  return response.data;
// └─ backend-ийн хариуны зөвхөн data хэсгийг буцаана
};
```

---

## ТАВДУГААР ХЭСЭГ: Нийт бүтэц

```
Хэрэглэгч товч дарна
       ↓
[LoginForm.tsx]
form.handleSubmit → validation → onSubmit(data)
       ↓
[useLoginForm.ts]
login(data) дуудна
       ↓
[providers.tsx]
loginMutation.mutateAsync(data) → loginUser(data) дуудна
       ↓
[auth.service.ts]
api.v1.authControllerLoginV1(data) дуудна
       ↓
[api.ts - автоматаар үүссэн]
HTTP request объект үүснэ: { method: POST, url: /v1/auth/login, body: data }
       ↓
[interceptor.ts]
interceptor-той axios instance request-г явуулна
       ↓
[Network - интернэт]
POST http://localhost:3001/v1/auth/login илгээнэ
       ↓
[Backend - NestJS localhost:3001]
Хүлээн авч, шалгаж, хариу буцаана
       ↓
[Network - интернэт буцаана]
200 OK + Set-Cookie эсвэл 401 Unauthorized
       ↓
[interceptor.ts]
401 бол → /login redirect
200 бол → response-г дамжуулна
       ↓
[auth.service.ts]
response.data буцаана
       ↓
[providers.tsx]
onSuccess → invalidateQueries → getMe() → user state шинэчлэгдэнэ
       ↓
[router.tsx]
isAuthenticated = true → ProtectedRoute → HomePage
```

# Календарийн Frontend ↔ Backend холболт
## Intern хөгжүүлэгчдэд зориулсан — Эхнээс нь дэлгэрэнгүй тайлбар

---

## Эхлэхийн өмнө — "HTTP Request" гэж юу вэ?

Frontend (browser) болон backend (сервер) нь интернэтээр ярилцдаг. Энэ ярилцлага нь **HTTP request/response** гэж нэрлэгддэг.

```
Browser                          Сервер (backend)
  │                                    │
  │  ──── GET /v1/calendar ──────────► │  "Энэ хэрэглэгчийн calendar мэдээлэл өгөөч"
  │                                    │
  │  ◄─── { days: [...] } ────────────  │  "Тийм ийм мэдээлэл байна"
  │                                    │
```

Манай кодын **гол зорилго** бол энэ ярилцлагыг зохион байгуулж, хүлээн авсан өгөгдлийг хэрэглэгчид харуулах явдал юм.

---

## Код ямар дарааллаар ажилладаг вэ?

Хэрэглэгч **нэг өдрийг дарах** үед дараах 7 алхам явагддаг:

```
1. HomePage.tsx         ← Хэрэглэгч өдрийг дарна
       ↓
2. useCalendar.ts       ← Hook идэвхждэг
       ↓
3. calendar.service.ts  ← API дуудлага бэлддэг
       ↓
4. api.ts               ← HTTP request илгээдэг
       ↓
5. interceptor.ts       ← Cookie нэмэх, 401 шалгах
       ↓
6. Backend сервер       ← Мэдээллийг боловсруулна
       ↓
7. Дахин буцаж ирнэ: api.ts → service → hook → HomePage → хэрэглэгч харна
```

Одоо **алхам бүрийг нэг нэгээр** тайлбарлая.

---

## АЛХАМ 1 — `HomePage.tsx` (Хэрэглэгч дарна)

**Файл:** `src/pages/home/HomePage.tsx`

```typescript
export default function HomePage() {
  // useState — React-ийн "санах" механизм
  // selectedDate нь одоогоор сонгосон огноо
  // setSelectedDate нь тэр огноог өөрчлөх функц
  const [selectedDate, setSelectedDate] = useState<Date | undefined>(
    new Date()  // ← эхлэхдээ өнөөдрийн огноо
  );

  // useCalendar hook-оос 3 зүйл авна:
  // - incomeDates: орлоготой өдрүүд (ногоон цэг харуулахад)
  // - withdrawDates: зарлагатай өдрүүд (улаан цэг харуулахад)
  // - dailyData: сонгосон өдрийн дэлгэрэнгүй мэдээлэл
  const { incomeDates, withdrawDates, dailyData } = useCalendar(selectedDate);

  return (
    <div>
      <CustomCalendar
        selected={selectedDate}
        onSelect={setSelectedDate}   // ← хэрэглэгч дарах үед setSelectedDate дуудагдана
        incomeDates={incomeDates}    // ← ногоон цэгтэй өдрүүд
        withdrawDates={withdrawDates} // ← улаан цэгтэй өдрүүд
      />

      <TransactionSummary data={dailyData} />  {/* ← сонгосон өдрийн жагсаалт */}
    </div>
  );
}
```

**Энд болдог зүйл:**
- `selectedDate` өөрчлөгдөхөд React автоматаар `useCalendar(selectedDate)` дахин ажиллуулна
- `useCalendar` дотроос шинэ API дуудлага явна
- Шинэ өгөгдөл ирэхэд UI автоматаар шинэчлэгдэнэ

---

## АЛХАМ 2 — `CustomCalendar.tsx` (Ногоон/улаан цэг харуулах)

**Файл:** `src/components/common/CustomCalendar.tsx`

```typescript
export const CustomCalendar = ({
  selected,         // ← одоогоор сонгосон өдөр
  onSelect,         // ← хэрэглэгч өдөр дарахад дуудагдах функц
  incomeDates,      // ← ногоон цэг гарах өдрүүдийн жагсаалт
  withdrawDates,    // ← улаан цэг гарах өдрүүдийн жагсаалт
}) => {
  return (
    <Calendar
      modifiers={{
        income: incomeDates,    // ← "эдгээр өдрүүдэд 'income' тэмдэглэгээ нэм"
        withdraw: withdrawDates, // ← "эдгээр өдрүүдэд 'withdraw' тэмдэглэгээ нэм"
      }}
      components={{
        DayButton: ({ modifiers, ...props }) => {
          const hasIncome = !!modifiers?.income;    // энэ өдөр орлого байна уу?
          const hasWithdraw = !!modifiers?.withdraw; // энэ өдөр зарлага байна уу?
          return (
            <div>
              <CalendarDayButton {...props} />
              {/* Доор нь цэг харуулна */}
              {hasIncome && <span className="bg-emerald-500" />}   {/* ногоон */}
              {hasWithdraw && <span className="bg-rose-500" />}    {/* улаан */}
            </div>
          );
        },
      }}
    />
  );
};
```

**Жишээ:** `incomeDates = [Date("2026-06-05"), Date("2026-06-17")]` байвал зөвхөн тэр 2 өдөрт ногоон цэг харагдана.

---

## АЛХАМ 3 — `useCalendar.ts` (Бүх зүйлийг нэгтгэх hook)

**Файл:** `src/pages/home/hooks/useCalendar.ts`

Энэ файл бол **хамгийн чухал файл**. Энд:
- Backend-с өгөгдөл татна
- Өгөгдлийг боловсруулна
- UI-д бэлэн болгоно

```typescript
import { useQuery } from "@tanstack/react-query";
import { format, startOfMonth, endOfMonth } from "date-fns";
import { getCalendar, getDayTransactions } from "@/services/calendar.service";
import { getCategories } from "@/services/category.service";
import { processTransactionData } from "@/libs/transformsTransaction";
import { QUERY_KEY } from "@/constants/queryKeys";
import type { Transaction } from "@/types/transaction.types";
import type { TransactionResponse } from "@/libs/api";

export const useCalendar = (selectedDate: Date | undefined) => {
```

### Алхам 3.1 — Огноог string болгон хөрвүүлэх

```typescript
  const displayDate = selectedDate ?? new Date();
  // ?? гэдэг нь "selectedDate байхгүй бол new Date() ашигла" гэсэн утга

  const year = displayDate.getFullYear();   // жишээ: 2026
  const month = displayDate.getMonth();     // жишээ: 5 (6-р сар, 0-оос эхэлдэг!)

  // "2026-06-01" хэлбэрт хөрвүүлэх
  const startDate = format(startOfMonth(displayDate), "yyyy-MM-dd");
  // startOfMonth(June 17) → June 1
  // format(June 1, "yyyy-MM-dd") → "2026-06-01"

  const endDate = format(endOfMonth(displayDate), "yyyy-MM-dd");
  // endOfMonth(June 17) → June 30
  // format(June 30, "yyyy-MM-dd") → "2026-06-30"

  const formattedDay = selectedDate
    ? format(selectedDate, "yyyy-MM-dd")  // "2026-06-17"
    : null;
```

**Яагаад string болгох вэ?** Backend нь `Date` объект биш, `"2026-06-17"` гэх мэт string хүлээн авдаг.

### Алхам 3.2 — Сарын overview татах (1-р API дуудлага)

```typescript
  const calendarQuery = useQuery({
    queryKey: QUERY_KEY.CALENDAR(year, month),
    // QUERY_KEY.CALENDAR(2026, 5) → ["calendar", 2026, 5]
    // Энэ "нэр" нь cache-ийн түлхүүр — ижил key байвал дахин fetch хийхгүй

    queryFn: () => getCalendar(startDate, endDate),
    // Жишээ: getCalendar("2026-06-01", "2026-06-30")
    // → GET /v1/calendar?startDate=2026-06-01&endDate=2026-06-30
  });
```

**Буцаж ирэх өгөгдөл:**
```json
{
  "days": [
    { "date": "2026-06-01", "income": 0,      "expense": 45000 },
    { "date": "2026-06-05", "income": 500000, "expense": 0     },
    { "date": "2026-06-17", "income": 200000, "expense": 12000 }
  ]
}
```

Зөвхөн **transaction байгаа өдрүүд** л жагсаалтанд байна. Transaction байхгүй өдрүүд байхгүй.

### Алхам 3.3 — Сонгосон өдрийн дэлгэрэнгүй татах (2-р API дуудлага)

```typescript
  const dayQuery = useQuery({
    queryKey: QUERY_KEY.CALENDAR_DAY(formattedDay ?? ""),
    // QUERY_KEY.CALENDAR_DAY("2026-06-17") → ["calendar", "day", "2026-06-17"]

    queryFn: () => getDayTransactions(formattedDay!),
    // Жишээ: getDayTransactions("2026-06-17")
    // → GET /v1/calendar/day?date=2026-06-17

    enabled: !!formattedDay,
    // !! гэдэг нь boolean болгоно: null → false, "2026-06-17" → true
    // enabled: false байвал useQuery API дуудлага хийхгүй!
    // Өдөр сонгоогүй бол дуудахгүй.
  });
```

**Буцаж ирэх өгөгдөл:**
```json
[
  {
    "id": 42,
    "userId": 1,
    "categoryId": 3,
    "type": "expense",
    "amount": 12000,
    "description": "Кофе",
    "transactionDate": "2026-06-17T09:30:00.000Z",
    "createdAt": "2026-06-17T09:31:00.000Z",
    "updatedAt": "2026-06-17T09:31:00.000Z"
  },
  {
    "id": 38,
    "userId": 1,
    "categoryId": 1,
    "type": "income",
    "amount": 200000,
    "description": "Цалин",
    "transactionDate": "2026-06-17T08:00:00.000Z",
    "createdAt": "2026-06-17T08:01:00.000Z",
    "updatedAt": "2026-06-17T08:01:00.000Z"
  }
]
```

### Алхам 3.4 — Категорийн нэр татах (3-р API дуудлага)

```typescript
  const categoriesQuery = useQuery({
    queryKey: QUERY_KEY.CATEGORIES,  // ["categories"]
    queryFn: getCategories,
    // → GET /v1/categories
  });
```

**Яагаад хэрэгтэй вэ?**

Transaction-д зөвхөн `categoryId: 3` гэж байдаг. Гэвч хэрэглэгчид "3" гэж харуулах биш, "Хоол" гэж харуулах хэрэгтэй. Тиймээс категорийн нэрүүдийг тусад нь татаад хөрвүүлдэг.

```typescript
  const categoryList = Array.isArray(categoriesQuery.data)
    ? categoriesQuery.data
    : [];
  // categoriesQuery.data нь undefined байж болох тул Array.isArray шалгана

  const categoryMap = new Map(categoryList.map((c) => [c.id, c.name]));
  // Map гэдэг нь { key → value } хадгалах JavaScript-ийн өгөгдлийн бүтэц
  // Жишээ: Map { 1 → "Цалин", 2 → "Хоол", 3 → "Кофе", ... }
  // categoryMap.get(3) → "Кофе"
```

### Алхам 3.5 — Ногоон/улаан цэгний өдрүүд гаргах

```typescript
  const incomeDates: Date[] = [];
  const withdrawDates: Date[] = [];

  (calendarQuery.data?.days ?? []).forEach((item) => {
    // item = { date: "2026-06-05", income: 500000, expense: 0 }
    const date = new Date(item.date);  // string → Date объект
    if (item.income > 0) incomeDates.push(date);   // ногоон цэг
    if (item.expense > 0) withdrawDates.push(date); // улаан цэг
  });
  // Нэг өдөрт хоёулаа байж болно (зөрүүлэн харуулна)
```

### Алхам 3.6 — TransactionResponse → Transaction хөрвүүлэлт

**Энэ бол хамгийн чухал хэсэг.** Backend-аас ирсэн өгөгдлийн shape болон UI-д хэрэгтэй shape ялгаатай:

```typescript
  // Backend-аас ирсэн (TransactionResponse — api.ts-д тодорхойлсон):
  // {
  //   id: 42,
  //   userId: 1,
  //   categoryId: 3,
  //   type: "expense",
  //   amount: 12000,
  //   description: "Кофе",         ← "description" нэртэй
  //   transactionDate: "2026-06-17T09:30:00.000Z",   ← бүтэн timestamp
  //   createdAt: "...",
  //   updatedAt: "..."
  // }

  // UI-д хэрэгтэй (Transaction — transaction.types.ts-д тодорхойлсон):
  // {
  //   id: 42,
  //   header: "Кофе",              ← "header" нэртэй болсон
  //   categoryId: 3,
  //   date: "2026-06-17",          ← зөвхөн огноо (цаг хасагдсан)
  //   amount: 12000,
  //   type: "expense",
  //   category: { id: 3, name: "Кофе" }  ← categoryId-г нэрээр нь баяжуулсан
  // }

  const mapToTransaction = (t: TransactionResponse): Transaction => ({
    id: t.id,
    header: t.description ?? "",
    //       t.description нь null байж болох тул ?? "" нэмсэн
    //       "Кофе" → "Кофе", null → ""
    categoryId: t.categoryId,
    date: t.transactionDate.slice(0, 10),
    //         "2026-06-17T09:30:00.000Z".slice(0, 10) → "2026-06-17"
    amount: t.amount,
    type: t.type,
    category: {
      id: t.categoryId,
      name: categoryMap.get(t.categoryId) ?? String(t.categoryId),
      //         categoryMap.get(3) → "Кофе"
      //         байхгүй бол String(3) → "3" (fallback)
    },
  });
```

### Алхам 3.7 — Мэдээллийг нэгтгэн боловсруулах

```typescript
  const dailyData = processTransactionData(
    (dayQuery.data ?? []).map(mapToTransaction)
    // dayQuery.data = [TransactionResponse, TransactionResponse, ...]
    // .map(mapToTransaction) = [Transaction, Transaction, ...]
    // processTransactionData(...) = нэгтгэсэн GroupedData объект
  );
```

`processTransactionData` функц нь transaction-уудыг авч нийт дүнг тооцно:
```
Input:  [{ type: "income", amount: 200000 }, { type: "expense", amount: 12000 }]

Output: {
  totalIncome: 200000,
  totalExpense: 12000,
  netBalance: 188000,
  incomes:  [{ id: 38, name: "Цалин", amount: 200000, description: "Цалин" }],
  expenses: [{ id: 42, name: "Кофе", amount: 12000, description: "Кофе" }]
}
```

### Алхам 3.8 — Hook-оос мэдээлэл буцааха

```typescript
  return {
    incomeDates,    // → CustomCalendar-д ногоон цэг харуулахад
    withdrawDates,  // → CustomCalendar-д улаан цэг харуулахад
    dailyData,      // → TransactionSummary-д дэлгэрэнгүй жагсаалт харуулахад
    isLoading: calendarQuery.isLoading,
    isDayLoading: dayQuery.isLoading,
  };
};
```

---

## АЛХАМ 4 — `calendar.service.ts` (HTTP дуудлага бэлдэх)

**Файл:** `src/services/calendar.service.ts`

```typescript
import { Api } from "@/libs/api";
import type { TransactionResponse } from "@/libs/api";
import axiosInstance from "@/libs/interceptor";

const api = new Api();       // api.ts-ийн Api классыг жишээ болгоно
api.instance = axiosInstance; // ← ЧУХАЛ: interceptor-той Axios-аар солино
                              // Үгүй бол cookie явахгүй, 401 redirect ажиллахгүй
```

### `getCalendar` функц

```typescript
export const getCalendar = async (
  startDate: string,  // "2026-06-01"
  endDate: string,    // "2026-06-30"
): Promise<CalendarResponse> => {

  const response = await api.v1.calendarControllerGetCalendarV1({
    startDate,  // shorthand: { startDate: startDate, endDate: endDate }
    endDate,
  });
  // Энэ нь дараах HTTP request илгээнэ:
  // GET https://api.example.com/v1/calendar?startDate=2026-06-01&endDate=2026-06-30
  // Cookie автоматаар нэмэгдэнэ (interceptor хариуцна)

  return response.data as unknown as CalendarResponse;
  // response.data нь { days: [...] } гэсэн объект
  // "as unknown as CalendarResponse" — TypeScript type cast
  // Яагаад хэрэгтэй вэ? api.ts-д энэ endpoint "void" буцаана гэж бичигдсэн
  // (Swagger-д response schema бичигдээгүй тул). Гэвч backend үнэндээ
  // { days: [...] } буцаадаг. Тиймээс TypeScript-ийг "итгэл" болгоод cast хийнэ.
};
```

### `getDayTransactions` функц

```typescript
export const getDayTransactions = async (
  date: string,  // "2026-06-17"
): Promise<TransactionResponse[]> => {

  const response = await api.v1.calendarControllerGetDayTransactionsV1({
    date,
  });
  // Илгээх request:
  // GET https://api.example.com/v1/calendar/day?date=2026-06-17

  return response.data as unknown as TransactionResponse[];
  // TransactionResponse нь api.ts-д байгаа type — дахин бичих шаардлагагүй!
  // Энэ бол auth.service.ts-тэй ижил pattern.
};
```

### Яагаад `CalendarDayTransaction` interface устгав?

Хуучнаар calendar.service.ts-д ингэж бичигдсэн байсан:
```typescript
// ❌ БУРУУ — шаардлагагүй давхардал
export interface CalendarDayTransaction {
  id: number;
  userId: number;
  categoryId: number;
  type: "income" | "expense";
  amount: number;
  description: string | null;
  transactionDate: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
}
```

Харин api.ts-д аль хэдийн байдаг:
```typescript
// ✅ api.ts-д байгаа — яг л ижил зүйл
export interface TransactionResponse {
  id: number;
  userId: number;
  categoryId: number;
  type: "income" | "expense";
  amount: number;
  description: string | null;
  transactionDate: string;
  createdAt: string;
  updatedAt: string;
}
```

`deletedAt` гэсэн нэмэлт field л байгаа ялгаа, гэвч backend нь устгагдаагүй transaction-уудыг л буцаадаг тул `deletedAt` нь бидэнд хэрэггүй. Тиймээс `CalendarDayTransaction`-г устгаж, `TransactionResponse` ашигласан.

---

## АЛХАМ 5 — `api.ts` (HTTP client)

**Файл:** `src/libs/api.ts`

Энэ файл нь **автоматаар үүсгэгддэг** — backend-ийн Swagger документациас:

```
Backend (NestJS)
  ↓ swagger.json үүсгэнэ
swagger-typescript-api хэрэгсэл
  ↓ хөрвүүлнэ
src/libs/api.ts (гараар засварлахгүй!)
```

Файлын дээд хэсэгт ингэж бичигдсэн:
```
## THIS FILE WAS GENERATED VIA SWAGGER-TYPESCRIPT-API
```

### `Api` классын бүтэц

```typescript
export class Api extends HttpClient {
  v1 = {
    // Calendar endpoints
    calendarControllerGetCalendarV1: (query: { startDate, endDate }) =>
      this.request({
        path: "/v1/calendar",
        method: "GET",
        query: query,
      }),

    calendarControllerGetDayTransactionsV1: (query: { date }) =>
      this.request({
        path: "/v1/calendar/day",
        method: "GET",
        query: query,
      }),

    // Auth endpoints
    authControllerLoginV1: (data: LoginDto) =>
      this.request({
        path: "/v1/auth/login",
        method: "POST",
        body: data,
      }),

    // ... бусад endpoint-үүд
  };
}
```

Жишээ нь `api.v1.calendarControllerGetCalendarV1({ startDate: "2026-06-01", endDate: "2026-06-30" })` дуудахад:
```
GET https://yourbackend.com/v1/calendar?startDate=2026-06-01&endDate=2026-06-30
```
гэсэн request явна.

---

## АЛХАМ 6 — `interceptor.ts` (Cookie & алдаа шалгах)

**Файл:** `src/libs/interceptor.ts`

```typescript
const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  // .env файлаас авна: VITE_API_BASE_URL=https://yourapi.com
  // Бүх request-ийн URL эхлэл болно

  withCredentials: true,
  // Cookie-г автоматаар бүх request-д нэмнэ
  // Үгүй бол backend "Хэн байгаа чи?" гэж мэдэхгүй
});

api.interceptors.response.use(
  (response) => response,
  // Амжилттай response → тэр чигт нь буцаана, өөр зүйл хийхгүй

  (error) => {
    if (
      error.response?.status === 401 &&
      window.location.pathname !== "/login"
    ) {
      window.location.href = "/login";
      // 401 = "Нэвтэрч ороогүй байна"
      // Login хуудсан дээр байхгүй бол автоматаар login руу шилжүүлнэ
    }
    return Promise.reject(error);
    // Алдааг дахин "шидэнэ" — useQuery-д isError: true болно
  },
);

export default api;
```

**Cookie гэж юу вэ?**

Browser нь backend-тай анхны удаа ярилцах үед backend нь cookie-д session мэдээллийг хадгалдаг. Дараагийн бүх request-д browser энэ cookie-г автоматаар илгээдэг. Ингэснээр backend нь "энэ хэрэглэгч нэвтэрсэн, userId нь 1" гэдгийг мэддэг.

---

## АЛХАМ 7 — Backend (сервер талд юу болдог вэ?)

**Файл:** `backend/src/calendar/calendar.service.ts`

### `getCalendar` — Сарын тойм

```typescript
async getCalendar(userId: number, dto: GetCalendarDto) {
  // userId нь cookie-с авна (interceptor-ийн тусламжтай)
  // dto = { startDate: "2026-06-01", endDate: "2026-06-30" }

  // SQL хийх утга:
  // SELECT DATE(transactionDate) as date,
  //        type,
  //        SUM(amount) as total
  // FROM transactions
  // WHERE userId = 1
  //   AND transactionDate >= '2026-06-01'
  //   AND transactionDate <= '2026-06-30'
  //   AND deletedAt IS NULL
  // GROUP BY DATE(transactionDate), type

  // Жишээ SQL үр дүн:
  // date         | type    | total
  // -------------|---------|-------
  // "2026-06-01" | expense | 45000
  // "2026-06-05" | income  | 500000
  // "2026-06-17" | income  | 200000
  // "2026-06-17" | expense | 12000

  // Дараа нь нэг өдөрт байгаа income/expense-ийг нэгтгэнэ:
  return {
    days: [
      { date: "2026-06-01", income: 0,      expense: 45000 },
      { date: "2026-06-05", income: 500000, expense: 0     },
      { date: "2026-06-17", income: 200000, expense: 12000 },
    ]
  };
}
```

### `getDayTransactions` — Өдрийн дэлгэрэнгүй

```typescript
async getDayTransactions(userId: number, date: string) {
  // "2026-06-17" → тухайн өдрийн 00:00:00 - 23:59:59 хооронд байгаа
  // бүх transaction-уудыг авна

  return [
    {
      id: 42, userId: 1, categoryId: 3,
      type: "expense", amount: 12000, description: "Кофе",
      transactionDate: "2026-06-17T09:30:00.000Z", ...
    },
    {
      id: 38, userId: 1, categoryId: 1,
      type: "income", amount: 200000, description: "Цалин",
      transactionDate: "2026-06-17T08:00:00.000Z", ...
    }
  ];
}
```

---

## Type-ийн зохион байгуулалт — Хаана юу байх ёстой вэ?

```
src/
├── libs/
│   └── api.ts              ← Backend-ийн бүх DTO/Response type (AUTO-GENERATED)
│                              LoginDto, RegisterDto, TransactionResponse,
│                              CategoryDto, CalendarResponse гэх мэт
│
├── types/
│   └── transaction.types.ts ← Frontend-ийн UI model-үүд
│                              Transaction, GroupedData, Category гэх мэт
│                              (backend-аас ирсэн өгөгдлийг UI-д тохируулсан)
│
└── services/
    ├── auth.service.ts      ← api.ts-аас import хийж ашиглана ✅
    ├── calendar.service.ts  ← api.ts-аас import хийж ашиглана ✅
    └── category.service.ts  ← api.ts-аас import хийж ашиглана ✅
```

### `api.ts` vs `transaction.types.ts` — Ялгаа юу вэ?

| | `api.ts` (TransactionResponse) | `transaction.types.ts` (Transaction) |
|---|---|---|
| **Хаанаас үүснэ** | Backend Swagger-аас auto-generate | Хөгжүүлэгч гараар бичнэ |
| **Юуг төлөөлнө** | Backend API-ийн яг response shape | UI-д харуулах shape |
| **`description` талбар** | `description: string \| null` | `header: string` (нэр өөр!) |
| **`transactionDate`** | `"2026-06-17T09:30:00.000Z"` (full) | `"2026-06-17"` (date only) |
| **`category`** | `categoryId: 3` (зөвхөн id) | `category: { id: 3, name: "Кофе" }` |

`mapToTransaction` функц нь зүүн баганыг баруун баганад хөрвүүлдэг.

---

## `queryKeys.ts` — Cache-ийн түлхүүрүүд

**Файл:** `src/constants/queryKeys.ts`

```typescript
export const QUERY_KEY = {
  USER: ["user"],
  CATEGORIES: ["categories"],
  CALENDAR: (year: number, month: number) => ["calendar", year, month],
  CALENDAR_DAY: (date: string) => ["calendar", "day", date],
};
```

TanStack Query нь `queryKey` ашиглан өгөгдлийг cache-д хадгалдаг. Ижил key байвал network request хийхгүй, cache-аас авдаг.

```
Хэрэглэгч 6-р сарын calendar нээнэ
→ queryKey: ["calendar", 2026, 5]
→ Cache-д байхгүй → API дуудлага хийнэ → cache-д хадгална

Хэрэглэгч 7-р сар руу шилжинэ, дахин 6-р сар руу ирнэ
→ queryKey: ["calendar", 2026, 5]
→ Cache-д БАЙНА → API дуудлага ХИЙХГҮЙ → cache-аас шууд харуулна
```

---

## Бүхэл бүтэн урсгал — Нэг жишээгээр

**Хэрэглэгч 2026 оны 6-р сарын 17-г дарав:**

```
1. HomePage.tsx
   setSelectedDate(Date("2026-06-17"))
   → useState өөрчлөгдөнө
   → React useCalendar дахин дуудна

2. useCalendar.ts
   startDate = "2026-06-01"
   endDate   = "2026-06-30"
   formattedDay = "2026-06-17"

3. calendarQuery (сарын overview)
   queryKey: ["calendar", 2026, 5]
   Cache байхгүй → getCalendar("2026-06-01", "2026-06-30") дуудна

4. calendar.service.ts → getCalendar()
   api.v1.calendarControllerGetCalendarV1({ startDate, endDate })

5. api.ts → HttpClient
   GET /v1/calendar?startDate=2026-06-01&endDate=2026-06-30

6. interceptor.ts
   Cookie нэмнэ: Cookie: session=abc123
   Request явна

7. Backend хүлээн авна
   userId = 1 (cookie-аас)
   SQL query ажиллана
   Response буцаана: { days: [...] }

8. Буцаж frontend-д ирнэ
   calendarQuery.data = { days: [
     { date: "2026-06-17", income: 200000, expense: 12000 },
     ...
   ]}

9. dayQuery (17-ний дэлгэрэнгүй)
   enabled: true (formattedDay байна)
   getDayTransactions("2026-06-17") дуудна
   GET /v1/calendar/day?date=2026-06-17
   Backend буцаана: [{ id:42, amount:12000, ... }, { id:38, amount:200000, ... }]

10. categoriesQuery
    getCategories() → GET /v1/categories
    [{ id:1, name:"Цалин" }, { id:3, name:"Кофе" }, ...]

11. useCalendar.ts боловсруулна
    categoryMap = Map { 1→"Цалин", 3→"Кофе" }
    mapToTransaction дуудаж хөрвүүлнэ:
      { id:42, header:"Кофе", category:{id:3,name:"Кофе"}, amount:12000, ... }
      { id:38, header:"Цалин", category:{id:1,name:"Цалин"}, amount:200000, ... }
    processTransactionData → { totalIncome:200000, totalExpense:12000, ... }

12. Homepage.tsx хүлээн авна
    incomeDates:   [Date("2026-06-05"), Date("2026-06-17")]
    withdrawDates: [Date("2026-06-01"), Date("2026-06-17")]
    dailyData:     { totalIncome:200000, totalExpense:12000, incomes:[...], expenses:[...] }

13. Харагдах байдал:
    - 17-ний дээр ногоон+улаан цэг
    - Баруун талд: орлого 200,000₮ / зарлага 12,000₮ харагдана
```

---

## Нийтлэг алдаа ба шийдэл

**1. "Cannot find name 'CalendarDayTransaction'"**
→ Уг interface-ийг calendar.service.ts-аас устгасан. Оронд нь `TransactionResponse` from `@/libs/api` import хий.

**2. API дуудлага явж байна гэхдээ өгөгдөл харагдахгүй**
→ Browser DevTools → Network tab → `/v1/calendar` гэж хайж response харна.

**3. 401 алдаа гарсаар байвал**
→ Нэвтрэлт тасарсан. Interceptor.ts login руу шилжүүлнэ. Session cookie дууссан байж болно.

**4. `as unknown as CalendarResponse` гэж яагаад хийдэг вэ?**
→ api.ts-д calendar endpoint-ийн return type нь `void` (Swagger-д response schema бичигдээгүй). Гэвч backend `{ days: [...] }` буцаадаг. TypeScript-ийн strict check-ийг тойрч гарахын тулд cast хийдэг. Backend Swagger-д response schema нэмэгдвэл cast хэрэггүй болно.

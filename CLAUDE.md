# CLAUDE.md — Ecommerce Website (Storefront)

Kontekst API backendu: `.claude/frontend/website/api-context.md`  
Kontekst API (pełny, z admin): `.claude/frontend/api-context.md`

---

## Stack

| Warstwa      | Biblioteka                                |
| ------------ | ----------------------------------------- |
| Framework    | Next.js 15 (App Router) + TypeScript      |
| Styling      | Tailwind CSS v4 + shadcn/ui               |
| Server state | TanStack Query v5 (dla Client Components) |
| Formularze   | React Hook Form + Zod                     |
| HTTP         | axios (jedna instancja per środowisko)    |
| Ikony        | lucide-react                              |
| Animacje     | Framer Motion (opcjonalnie)               |

> **Dlaczego Next.js 15 App Router?** Strony produktów i kategorii wymagają SSR/SSG dla SEO. Server Components pobierają dane bezpośrednio — brak waterfall, brak spinnerów na first load.

---

## Struktura projektu

```
src/
  app/                          ← Next.js App Router
    (store)/                    ← Route group — layout sklepu (nagłówek, stopka, koszyk)
      layout.tsx
      page.tsx                  ← Strona główna
      products/
        page.tsx                ← Lista produktów + filtry
        [slug]/
          page.tsx              ← Strona produktu
      categories/
        [slug]/
          page.tsx              ← Produkty w kategorii
      brands/
        [slug]/
          page.tsx              ← Produkty marki
      cart/
        page.tsx                ← Strona koszyka
      checkout/
        page.tsx                ← Formularz zamówienia
      account/
        layout.tsx              ← Guard — tylko zalogowani
        page.tsx                ← Mój profil
        addresses/
          page.tsx              ← Moje adresy
        orders/
          page.tsx              ← Historia zamówień (przyszłość)
    (auth)/                     ← Route group — bez layoutu sklepu
      login/
        page.tsx
      register/
        page.tsx
      forgot-password/
        page.tsx
      reset-password/
        page.tsx
    api/                        ← Next.js Route Handlers (opcjonalnie dla proxy)
  components/
    ui/                         ← shadcn/ui (nie edytować ręcznie)
    layout/
      Header.tsx
      Footer.tsx
      CartDrawer.tsx            ← Wysuwana szuflada koszyka
    product/
      ProductCard.tsx
      ProductGallery.tsx
      ProductVariantPicker.tsx
      AddToCartButton.tsx
    cart/
      CartItem.tsx
      CartSummary.tsx
    catalog/
      ProductFilters.tsx
      ProductGrid.tsx
      SortSelect.tsx
  features/
    auth/
      hooks/                    ← useAuth, useLogin, useRegister
      AuthProvider.tsx          ← Context + token management
    cart/
      hooks/                    ← useCart, useAddToCart, useUpdateCartItem, useRemoveCartItem
      CartProvider.tsx          ← Optymistyczne aktualizacje + guest cart sync
    catalog/
      hooks/                    ← useProducts, useProduct, useCategories
  lib/
    api/
      client.ts                 ← axios instance
      auth.ts                   ← auth endpoints + typy
      catalog.ts                ← catalog endpoints + typy
      cart.ts                   ← cart endpoints + typy
      users.ts                  ← user/address endpoints + typy
    utils.ts                    ← cn(), formatPrice(), formatDate()
    validators.ts               ← wspólne schematy Zod
  middleware.ts                 ← ochrona /account/* routes
```

---

## Kluczowe wzorce

### Server Components (domyślne w App Router)

Używaj dla stron które mają być indeksowane lub potrzebują szybkiego first load:

```typescript
// app/(store)/products/[slug]/page.tsx
export default async function ProductPage({ params }: { params: { slug: string } }) {
  const product = await getProduct(params.slug);   // bezpośredni fetch, nie przez TanStack Query
  if (!product) notFound();

  return (
    <>
      <ProductGallery images={product.images} />
      <ProductVariantPicker product={product} />    {/* Client Component dla interakcji */}
    </>
  );
}

export async function generateMetadata({ params }: { params: { slug: string } }) {
  const product = await getProduct(params.slug);
  return { title: product?.name, description: product?.shortDescription };
}
```

### Client Components — tylko gdy potrzebujesz

Oznacz `'use client'` wyłącznie dla komponentów które:

- używają `useState` / `useEffect`
- obsługują zdarzenia użytkownika (klik, submit)
- używają TanStack Query

```typescript
'use client';
// components/product/AddToCartButton.tsx
export function AddToCartButton({ variantId, productId, productName, variantName, imageUrl }: Props) {
  const { mutate, isPending } = useAddToCart();

  return (
    <Button onClick={() => mutate({ productId, variantId, quantity: 1, productName, variantName, imageUrl })}
            disabled={isPending}>
      {isPending ? 'Dodawanie...' : 'Dodaj do koszyka'}
    </Button>
  );
}
```

### Pliki API (`lib/api/`)

```typescript
// lib/api/catalog.ts
export interface ProductListItem {
  id: string;
  name: string;
  slug: string;
  mainCategoryId: string;
  brandId: string | null;
  isFeatured: boolean;
  tags: string[];
  createdAt: string;
}

export interface ProductDto {
  id: string;
  name: string;
  slug: string;
  description: string | null;
  shortDescription: string | null;
  status: "Draft" | "Active" | "Inactive" | "Archived";
  variants: ProductVariantDto[];
  images: ProductImageDto[];
  attributes: ProductAttributeValueDto[];
  // ...
}

export async function getProduct(slug: string): Promise<ProductDto | null> {
  try {
    const { data } = await apiClient.get(`/catalog/products/${slug}`);
    return data;
  } catch (e: any) {
    if (e?.response?.status === 404) return null;
    throw e;
  }
}
```

### Koszyk — kluczowa logika

```typescript
// features/cart/CartProvider.tsx
'use client';

// Koszyk gościa: localStorage['guestCart'] = CartItem[]
// Po zalogowaniu: POST /cart/merge → re-fetch GET /cart → clear localStorage

export function CartProvider({ children }: { children: React.ReactNode }) {
  const { data: session } = useAuth();

  // Scal koszyk gościa przy logowaniu
  useEffect(() => {
    if (!session?.userId) return;
    const guestCart = getGuestCart();           // localStorage
    if (guestCart.length > 0) {
      mergeCart({ items: guestCart })
        .then(() => clearGuestCart())
        .then(() => queryClient.invalidateQueries({ queryKey: ['cart'] }));
    }
  }, [session?.userId]);

  return <>{children}</>;
}
```

```typescript
// features/cart/hooks/useCart.ts
"use client";

export function useCart() {
  const { data: session } = useAuth();

  return useQuery({
    queryKey: ["cart"],
    queryFn: () => getCart(),
    enabled: !!session, // fetch tylko dla zalogowanych
    staleTime: 30_000,
  });
}

export function useAddToCart() {
  const queryClient = useQueryClient();
  const { data: session } = useAuth();

  return useMutation({
    mutationFn: (item: AddCartItemRequest) =>
      session ? addCartItem(item) : addToGuestCart(item), // guest → localStorage
    onSuccess: () => {
      if (session) queryClient.invalidateQueries({ queryKey: ["cart"] });
    },
  });
}
```

---

## Autentykacja

- Access token: zmienna modułu w `lib/api/client.ts` (pamięć, nie localStorage — bezpieczniej)
- Refresh token: `localStorage['refreshToken']`
- Interceptor axios: przy 401 → `POST /auth/refresh` → ponów; przy błędzie refresh → wyloguj
- Middleware Next.js (`middleware.ts`): sprawdza cookie `isLoggedIn` (nie JWT!) do ochrony tras

```typescript
// middleware.ts — lekka ochrona tras (pełna weryfikacja na serwerze)
export function middleware(request: NextRequest) {
  const isLoggedIn = request.cookies.get("isLoggedIn")?.value === "true";
  if (!isLoggedIn && request.nextUrl.pathname.startsWith("/account")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
}
export const config = { matcher: ["/account/:path*"] };
```

```typescript
// lib/api/client.ts
let accessToken: string | null = null;
export const setAccessToken = (t: string | null) => {
  accessToken = t;
};

// interceptor request
apiClient.interceptors.request.use((config) => {
  if (accessToken) config.headers.Authorization = `Bearer ${accessToken}`;
  return config;
});

// interceptor response — auto-refresh
apiClient.interceptors.response.use(
  (res) => res,
  async (error) => {
    const original = error.config;
    if (error.response?.status === 401 && !original._retry) {
      original._retry = true;
      try {
        const { data } = await apiClient.post("/auth/refresh", {
          token: localStorage.getItem("refreshToken"),
        });
        setAccessToken(data.accessToken);
        localStorage.setItem("refreshToken", data.refreshToken);
        return apiClient(original);
      } catch {
        setAccessToken(null);
        localStorage.removeItem("refreshToken");
        window.location.href = "/login";
      }
    }
    return Promise.reject(error.response?.data ?? error);
  },
);
```

---

## Ceny — reguły wyświetlania

Backend nigdy nie zwraca ceny na `ProductListItem` (pole `price` zawsze `null`).  
Cena pochodzi wyłącznie z wariantów w `ProductDto`.

```typescript
// Wyświetlanie ceny na karcie produktu — pobierz domyślny wariant
const defaultVariant =
  product.variants.find((v) => v.isDefault) ?? product.variants[0];
const price = defaultVariant?.price;
const compareAtPrice = defaultVariant?.compareAtPrice;

// Formatowanie
const formatPrice = (price: number) =>
  new Intl.NumberFormat("pl-PL", { style: "currency", currency: "PLN" }).format(
    price,
  );
```

---

## SEO

```typescript
// app/(store)/products/[slug]/page.tsx
export async function generateMetadata({ params }) {
  const product = await getProduct(params.slug);
  return {
    title: product?.name,
    description:
      product?.shortDescription ?? product?.description?.slice(0, 160),
    openGraph: {
      images:
        product?.images.filter((i) => !i.variantId).map((i) => i.url) ?? [],
    },
  };
}

// Statyczne ścieżki dla popularnych produktów
export async function generateStaticParams() {
  const { items } = await getProducts({ page: 1, pageSize: 100 });
  return items.map((p) => ({ slug: p.slug }));
}
```

---

## Routing

```
/                         → Strona główna (wyróżnione, kategorie, bestsellery)
/products                 → Lista wszystkich produktów + filtry + sortowanie
/products/{slug}          → Strona produktu
/categories/{slug}        → Produkty w kategorii
/brands/{slug}            → Produkty marki
/cart                     → Koszyk
/checkout                 → Zamówienie (wymaga zalogowania)
/account                  → Profil (wymaga zalogowania)
/account/addresses        → Adresy dostawy
/login                    → Logowanie
/register                 → Rejestracja
/forgot-password          → Reset hasła
/reset-password           → Ustaw nowe hasło (z tokenem z emaila)
```

---

## Zmienne środowiskowe

```env
NEXT_PUBLIC_API_URL=http://localhost:5000/api
```

---

## Polecenia

```bash
npm run dev          # dev server (http://localhost:3000)
npm run build        # build produkcyjny
npm run start        # serwer produkcyjny
npm run typecheck    # tsc --noEmit
npm run lint         # eslint
```

---

## Non-negotiable rules

- **Server Components domyślnie** — `'use client'` tylko gdy niezbędne (interakcje, hooki)
- **Nigdy nie edytuj `src/components/ui/`** — generowane przez shadcn CLI
- **Cena zawsze z wariantów** — `product.price` jest zawsze `null`
- **Koszyk gościa w localStorage** — scal przez `POST /cart/merge` przy logowaniu
- **Nie cachuj cen** — `currentPrice` w `CartItemDto` jest live, odśwież po powrocie
- **`generateMetadata` na każdej stronie** — SEO jest priorytetem dla sklepu
- Nie używaj `any` — `unknown` + zawęź typ

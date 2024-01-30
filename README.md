# Buscando dados com Next.js

## Neste capítulo...
1. Como usar SQL, APIs e ORMs com Next.js 

2. Como Server Components podem ajudar você a acessar recursos back-end com mais segurança.
3. Como implementar busca de dados paralelas usando JavaScript Pattern

---

# Consultas do Banco de dados (SQL)

😀 **Observação:** `PostgreSQL` é usado para escrever a lógica do Banco de dados SQL para agir na Aplicação Full-Stack. É possível fazer isso com `ORMs` como o `Prisma`.

## Casos onde usarei Consultas de Banco de dados **no Next.js**:
- Quando escrever uma `API` irei precisar escrever lógica para interagir com o Banco de dados.

- Se eu estiver usando o ReactJS com Server Components (buscando dados no servidor), eu poderei pular a etapa da API, e consultar a base de dados diretamente sem riscos de expor os segredos da base de dados ao cliente.

---

# Usando `Server Components` para buscar dados

Por padrão o Next.js usa Server Components. Há alguns benefícios em usá-los com o Next.js:

- Usar `async/await` para buscar dados sem entrar em contato com bibliotecas como `useState` e `useEffect`.

- Server Components são executados no servidor, para que possa manter a busca de dados e a lógica no servidor, e entregar somente os dados ao cliente.

- Desde que Server Components são armazenados nos Servidores, os dados podem ser consultados diretamente sem o uso de uma API!


--- 

# Prática: Usando o SQL!

Dentro de `/app/lib/data.ts` iremos **importar a função `sql` de `@vercel/postgres`**. Essa função faz com que eu consiga consultar a Base de dados.

```javascript
import { sql } from '@vercel/postgres';
```

😁 **Bom saber:** Pode chamar o recurso `sql` dentro de qualquer Server Component. Mas para navegar entre os componentes com mais facilidade, mantenha todas as consultas dentro de `/app/lib/data.ts` e importe-as dentro do componente onde queira mostrar os dados.

---

## Buscando dados para a página de Visão Geral do Dashboard (`/app/dashboard/page.tsx`) 

### Entenda o código abaixo.

```javascript

import { Card } from '@/app/ui/dashboard/cards';
import RevenueChart from '@/app/ui/dashboard/revenue-chart';
import LatestInvoices from '@/app/ui/dashboard/latest-invoices';
import { lusitana } from '@/app/ui/fonts';
 
export default async function Page() {
  return (
    <main>
      <h1 className={`${lusitana.className} mb-4 text-xl md:text-2xl`}>
        Dashboard
      </h1>
      <div className="grid gap-6 sm:grid-cols-2 lg:grid-cols-4">
        { <Card title="Collected" value={totalPaidInvoices} type="collected" /> }
        { <Card title="Pending" value={totalPendingInvoices} type="pending" /> }
        { <Card title="Total Invoices" value={numberOfInvoices} type="invoices" />}
        { <Card
          title="Total Customers"
          value={numberOfCustomers}
          type="customers"
        /> }
      </div>
      <div className="mt-6 grid grid-cols-1 gap-6 md:grid-cols-4 lg:grid-cols-8">
        {<RevenueChart revenue={revenue}  />}
        {<LatestInvoices latestInvoices={latestInvoices} />}
      </div>
    </main>
  );
}
```
### Observe que:

- No código acima, é mostrando que esse é um componente com função assíncrona, o que mostra que usarei o `await` para buscar os dados. 
- Há 3 componentes que recebem dados: `<Card/>`, `<RevenueChart/>` e `<LatestInvoices/>`. 

### Buscando dados para `<RevenueChart/>`

Para buscar dados para o componente `<RevenueChart/>`, importe a função `fetchRevenue` de `/app/lib/data.ts` para dentro do componente do Dashboard. 

**(Lembre-se: todos as consultas e lógica do banco de dados vêm de dentro de `/app/lib/data.ts`. Nesse caso a função `fetchRevenue` é a função que possui todas as consultas SQL e lógica para os dados que serão mostrados dentro do compontente `<RevenueChart/>`).**

**Exemplo da importação, abaixo:**

```javascript
import { Card } from '@/app/ui/dashboard/cards';
import LatestInvoices from '@/app/ui/dashboard/latest-invoices';
import { lusitana } from '@/app/ui/fonts';

//Abaixo, o componente para o qual os dados irão.
import RevenueChart from '@/app/ui/dashboard/revenue-chart';
//Abaixo, a função de onde os dados virão.
import { fetchRevenue } from '@/app/lib/data';
 
export default async function Page() {
  const revenue = await fetchRevenue();
  // ...
}
```
### Contudo... Há uma coisa que você precisa estar ciente:

  1. As solicitações de dados estão bloqueando involuntariamente umas às outras, criando uma `cascata de solicitações` `(request waterfalls)`.

## O que são `Request Waterfalls`?

Uma "request waterfall" refere-se a uma sequência de solicitações de rede que dependem da conclusão de solicitações anteriores. No caso de busca de dados, cada solicitação só pode começar depois que a solicitação anterior tiver retornado dados.

Esse padrão não é necessariamente ruim. Pode haver casos em que você quer request waterfall porque você deseja que uma condição seja satisfeita antes de fazer a próxima solicitação. Durante Por exemplo, talvez você queira buscar o ID e as informações de perfil de um usuário primeiro. Uma vez você tem o ID, você pode então prosseguir para buscar sua lista de amigos. Neste caso, Cada solicitação depende dos dados retornados da solicitação anterior.

No entanto, esse comportamento também pode ser não intencional e afetar o desempenho.

---

# Busca de dados paralelos (Parallel data fetching)

Uma maneira comum de evitar cascatas é iniciar todas as solicitações de dados ao mesmo tempo - em paralelo.

Em JavaScript, você pde usar as funções `Promise.all()` ou `Promise.allSettled()` pra iniciar todas as promessas ao mesmo tempo. Por exemplo em `/app/lib/data.ts`, foi usado `Promise.all()` na função `fetchCardData()`, exemplo abaixo:

```javascript
export async function fetchCardData() {
  try {
    const invoiceCountPromise = sql`SELECT COUNT(*) FROM invoices`;
    const customerCountPromise = sql`SELECT COUNT(*) FROM customers`;
    const invoiceStatusPromise = sql`SELECT
         SUM(CASE WHEN status = 'paid' THEN amount ELSE 0 END) AS "paid",
         SUM(CASE WHEN status = 'pending' THEN amount ELSE 0 END) AS "pending"
         FROM invoices`;
 
    //USado aqui abaixo.
    const data = await Promise.all([
      invoiceCountPromise,
      customerCountPromise,
      invoiceStatusPromise,
    ]);
    // ...
  }
}
```

Usando esse padrão, você pode:

- Começar a executar todas as buscas de dados ao mesmo tempo, o que pode levar a ganhos de desempenho.

- Use um padrão JavaScript nativo que possa ser aplicado a qualquer biblioteca ou estrutura.

No entanto, há uma **desvantagem** de confiar apenas nesse padrão JavaScript: o que acontece se uma solicitação de dados for mais lenta do que todas as outras?

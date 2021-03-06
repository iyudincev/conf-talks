# DataLoader — правильно решаем проблему N+1 запросов

[DataLoader](https://github.com/facebook/dataloader) - это утилита которая позволяет вам сократить кол-во запросов в базу данных через batching.

Давайте смоделируем ситуацию для более ясного представления проблемы. К примеру у нас есть GraphQL-запрос в котором мы получаем список статей из 15 элементов, и для каждой статьи получаем имя автора. Такой GraphQL-запрос может выглядеть следующим образом:

```graphql
{
  articles {
    title
    author {
      name
    }
  }
}
```

Код GraphQL-схемы можно представить так:

```js
import { GraphQLSchema, GraphQLList, GraphQLObjectType, GraphQLString, GraphQLInt } from 'graphql';
import { articleModel, authorModel } from './data';

const AuthorType = new GraphQLObjectType({
  name: 'Author',
  fields: () => ({
    id: { type: GraphQLInt },
    name: { type: GraphQLString },
    email: { type: GraphQLString },
  }),
});

const ArticleType = new GraphQLObjectType({
  name: 'Article',
  fields: () => ({
    title: { type: GraphQLString },
    authorId: { type: GraphQLString },
    author: {
      type: AuthorType,
      resolve: source => {
        return authorModel.findById(source.authorId);
      },
    },
  }),
});

export default new GraphQLSchema({
  query: new GraphQLObjectType({
    name: 'Query',
    fields: {
      articles: {
        type: new GraphQLList(ArticleType),
        resolve: () => {
          return articleModel.findMany();
        },
      },
    },
  }),
});
```

Это довольно стандартная ситуация в мире GraphQL и на первый взгляд не содержит в себе никакого подвоха, пока мы не врубим логирования запросов к нашей базе данных. Для запроса, который возвращает 15 статей в логе сервера мы можем увидеть 16 запросов к базе:

```text
Run Article query: findMany()
Run Author query: findById(1)
Run Author query: findById(7)
Run Author query: findById(6)
Run Author query: findById(3)
Run Author query: findById(4)
Run Author query: findById(5)
Run Author query: findById(6)
Run Author query: findById(7)
Run Author query: findById(3)
Run Author query: findById(2)
Run Author query: findById(5)
Run Author query: findById(4)
Run Author query: findById(2)
Run Author query: findById(1)
Run Author query: findById(1)
```

Кто-то скажет: Фууу, какая отвратительная производительность! GraphQL лажа полная!

Но на самом деле в GraphQL скормили лажовый код в resolve-методах. О таком поведении GraphQL давно знают в Facebook'е. И в свое время они сильно поломали голову над ее решением. И решение было найдено и выкинуто в OpenSource вместе с GraphQL - это [DataLoader](https://github.com/facebook/dataloader).

## Как работают `resolve`-методы в GraphQL

Но прежде чем перейдем к DataLoader, давайте разберемся с самим GraphQL и почему он так себя ведет. Во-первых, вы должны четко понимать, что для каждого поля в GraphQL-ответе был вызван некий resolve-метод. Даже если вы явно для какого-то поля не указали `resolve`-метод, то используется стандартный метод, [который выглядит так](https://github.com/graphql/graphql-js/blob/f529809f5408fb9a343e669ea4d4851add3df004/src/execution/execute.js#L1224-L1238):

```js
export const defaultFieldResolver: GraphQLFieldResolver<any, *> = function(source, args, context, info) {
  if (typeof source === 'object' || typeof source === 'function') {
    const property = source[info.fieldName];
    if (typeof property === 'function') {
      return source[info.fieldName](args, context, info);
    }
    return property;
  }
};
```

Т.е. для нашего примера со статьями и именами авторов, вызываются следующие resolve-методы:

- `Query.articles.resolve()` - первый метод получения списка статей, который возвращает массив из 15 статей
- дальше для каждой из 15 записей Article, будет вызван метод `Article.author.resolve(source, args, info, context)`, где в качестве `source` передается объект конкретной статьи и мы берем `source.authorId` чтобы сделать запрос в базу для получения автора по Id.
- ну а потом для каждого из 15 полей `Author.name` вызывается дефолтный резолвер, чтоб из объекта `Author` получить значение для проперти `name`.

Наша проблема отправки кучи запросов в базу, лежит ровно в резолвере получения автора по Id. Чтоб ее избежать необходимо как-то собрать все айдишники необходимых авторов и одним запросом заполучить всех авторов. А потом как-то их обратно разложить по резолверам. Задача звучит как туго-решаемая. Но нам париться не нужно, именно этим и занимается `DataLoader`.

## Как работает DataLoader?

DataLoader — это batcher и cache в одном флаконе. Его внутреннее устройство до банальности просто и сценарий работы можно описать так:

- вы вызываете метод `load(1)`, он сразу возвращает объект Promise
- вы вызываете метод `load(2)`, он вернет другой промис для второго объекта
- снова вызовите `load(1)`, вернется существующий Promise из первого шага (сработал кэш по айдишнику 1)
- допустим дозапросили `load(10)`, `load(4)` и на каждый получили по новому промису
- дальше на `nextTick()`, когда текущий код отработает и стек вызовов закончится, будет вызвана `batch loading function`, в которую будут переданы все уникальные айдишники по которым вызывался метод `load` — `batchLoad([1, 2, 10, 4])` (id 1 будет передан только один раз)
- в `batchLoad` вы получили массив айдишников, и теперь должны выдрать данные из базы и вернуть `В ТОМ ЖЕ ПОРЯДКЕ` массив объектов. Т.е. должны вернуть `[{ ...author1 }, undefined, { ...author10 }, { ...author4 }]`, `undefined` возвращаем для ненайденной записи, чтоб не сломать порядок.
- после того, как DataLoader получил данные он начинает их резолвить по промисам из первых шагов. Промис полученный от вызова `load(1)` зарезолвится со значением `author1`.
- если вдруг дальше в коде произойдет повторный вызов `load(1)`, то будет возвращен закешированный зарезолвенный промис со значением `author1`. Повторного обращения к базе не будет.

В виде кода можно это все дело записать так:

```js
// Создаем объект DataLoader и сразу в конструктор передаем функцию batch-загрузки по ids
const authorDataLoader = new DataLoader(
  async batchLoad(ids: any) => {
    // получили массив айдишников, дергаем записи одним запросом из базы
    const rows = await authorModel.findByIds(ids);
    // ВАЖНО: полученные записи мы ДОЛЖНЫ вернуть в том порядке как получили ids
    // если запись по id не будет найдена, то вернется undefined
    const sortedInIdsOrder = ids.map(id => rows.find(x => x.id === id));
    return sortedInIdsOrder;
  }
);

// запрашиваем необходимые данные
const authorPromise1 = authorDataLoader.load(1);
const authorPromise2 = authorDataLoader.load(2);
const authorPromise1_ = authorDataLoader.load(1); // authorPromise1 === authorPromise1_
const authorPromise10 = authorDataLoader.load(10);
const authorPromise4 = authorDataLoader.load(4);

// ... дальше код который работает с промисами на nextTick()
// но прежде чем он начнет выполняться, отработает `batchLoad` функция
```

В сухом остатке: мы инициализировали объект `DataLoader` сразу передав ему `batchLoad`-функцию в конструктор, ну а потом в коде вызываем `load` методы, там где нам нужно. Все просто, пока не начнем женить всё это дело с GraphQL.

## Женим DataLoader с GraphQL

Это самая важная часть этой статьи — правильная женитьба DataLoader и GraphQL. Раньше я не понимал, что на этом этапе большинство народу "велосипедит", пока в [чате по GraphQL](https://t.me/graphql_ru) не прочитал кучу негатива в его сторону. Я понял, что народ не понимает как правильно с ним работать.

Первым делом залез в документашку по DataLoader и ужаснулся. Там реально нет толкового примера как правильно работать с GraphQL. А изобилие текста и всяких дополнительных методов уводит народ непонятно куда. Первым делом он начинает восприниматься как некий cache-механизм, как замена для Redis'а. Во-вторых, очень сложно уловить, что они должны храниться в `context`е GraphQL. В-третьих, что его надо объявлять один раз и юзать во всех резолверах.

DataLoader простой и тупой и должен решать только проблему N+1 Query. Не пытайтесь из него сделать серебряную пулю. Именно использование его в качестве серебряной пули делает ваш код просто ужасным.

### Правило #1

**DataLoader должен использоваться в рамках одного запроса.** Не пытайтесь в него складывать все записи из базы данных и в случае изменения, удалять старую запись из кеша. Если у вас одна нода на которой работает GraphQL-сервер, то еще это как-то будет работать. Но если изменения происходят на одной ноде, то инвалидировать данные на другой ноде становиться труднорешаемой задачей.

Как правильно использовать? При получении запроса создайте DataLoader в контексте GraphQL. Выполните запрос. Как запрос отработает, пусть Garbage Collector его очистит вместе с контекстом. Для новых запросов вы создадите новые DataLoader'ы.

### Правило #2

**Для каждого резолвера свой DataLoader.** Не пытайтесь создавать уникальных даталоадеров на все случаи жизни. Если в вашей схеме несколько мест, где вы получаете данные автора, то велик соблазн объявить один дата-лоадер загрузки авторов. Намерение очень похвальное и хорошее, но оно ведет к грязному коду.

Во-первых, чтобы объявить общий дата-лоадер, то у вас одно место где это можно сделать — на уровне формирования GraphQL-контекста. Контекст объявляется на уровне сервера и соответственно логика получения авторов по id (batch-функция) у вас будет лежать на неверном уровне абстракции. Вы инициализируете сервак и тут же пишете как получать авторов. Это криво, в будущем рефакторинг будет адским.

Во-вторых, общий дата-лоадер должен дергать из базы полную запись. Т.к. вы наперед не знаете какие именно данные потребуются в вашем GraphQL-запросе. Может нужно просто имя и аватарку запросить, а вы тянете из базы 100500 полей. Но если будете тянуть только имя и аватарку, то в какой-то момент сильно озадачитесь, а почему не возвращаются email и еще какое-то поле?! В общем, в разных местах вашей схемы в разных резолверах требуются разные данные, и написать универсальное решение превращается в сложную задачу.

Поэтому, DataLoader необходимо объявлять внутри resolve-метода. Просто, квадратно и эффективно. А самое главное - в коде чисто. В 99% случаев вам не нужны общие дата-лоадеры на уровне всей схемы; на уровне резолверов вполне достаточно. Причем в resolve-методе у вас есть возможность считать, какие поля запросил пользователь и только их дернуть из БД.

### Пример использования DataLoader на уровне GraphQL-резолверов

После того как мы с вами разобрали как работают GraphQL-резолверы и DataLoader; разобрали основные правила по которым должен использоваться DataLoader в связке с GraphQL; самое время переписать наш код.

Во-первых, на уровне сервера, когда мы формируем GraphQL-контектст, мы должны объявить `WeakMap`. Он позволит нам находить уже существующие дата-лоадары, а если нет, то создавать и ложить туда новый инстанс дата-лоадера для повторного использования:

```diff
const server = new ApolloServer({
  schema,
  context: ({ req }) => ({
    req,
+    dataloaders: new WeakMap(),
  }),
});
```

Ну а дальше всю логику получения автора через DataLoader запихнуть в resolver. Запихнуть в то место, где у нас и появляется проблема N+1 Query:

```js
const ArticleType = new GraphQLObjectType({
  name: 'Article',
  fields: () => ({
    title: { type: GraphQLString },
    authorId: { type: GraphQLString },
    author: {
      type: AuthorType,
      // БЫЛО ТАК:
      // resolve: source => {
      //   return authorModel.findById(source.authorId);
      // },

      // А С DATA-LOADER ДОЛЖНО БЫТЬ ТАК:
      resolve: (source, args, context, info) => {
        // context.dataloaders был создан на уровне сервера (см сниппет кода выше)
        const { dataloaders } = context;

        // единожды инициализируем DataLoader для получения авторов по ids
        let dl = dataloaders.get(info.fieldNodes);
        if (!dl) {
          dl = new DataLoader(async (ids: any) => {
            // обращаемся в базу чтоб получить авторов по ids
            const rows = await authorModel.findByIds(ids);
            // IMPORTANT: сортируем данные из базы в том порядке, как нам передали ids
            const sortedInIdsOrder = ids.map(id => rows.find(x => x.id === id));
            return sortedInIdsOrder;
          });
          // ложим инстанс дата-лоадера в WeakMap для повторного использования
          dataloaders.set(info.fieldNodes, dl);
        }

        // юзаем метод `load` из нашего дата-лоадера
        return dl.load(source.authorId);
      },
    },
  }),
});
```

По коду выше хочется остановиться на том, почему используется `WeakMap` и `info.fieldNodes`. Для каждого резолвера нам необходимо находить уже созданный дата-лоадер. И единственный способ определить является ли резолвер одним и тем же, это сравнить из четвертого аргумента `info` значения проперти `fieldNodes`. Т.к. он является объектом, то можно смело заюзать `WeakMap` где в качестве ключа будет использоваться объект `info.fieldNodes`, а в качестве значения наш DataLoader для получения автора.

Также здесь важно понять, что в `info.fieldNodes` прописан кусок GraphQL-запроса со списком необходимых полей для типа `Author`. Поэтому объявляя DataLoader на уровне резолвера, вы можете запросить только те поля из базы, которые запросил пользователь в своем GraphQL-запросе. И если в другом месте GraphQL-запроса используется этот же резолвер, то для него будет создан отдельный DataLoader (т.к. `info.fieldNodes` уже будет другим).

В общем, для нашего простого запроса из примера выше:

```graphql
{
  articles {
    title
    author {
      name
    }
  }
}
```

C использованием DataLoader'а мы получим всего 2 запроса, вместо 16ти как было раньше:

```text
Run Article query: findMany()
Run Author query: findByIds(1, 7, 6, 3, 4, 5, 2)
```

## В сухом остатке

DataLoader клевая утилита, если ее использовать разумно. Если к ней относиться как к простому группировщику запросов.

- Не пытайтесь из DataLoader'а смастерить кэш
- Создавайте новые DataLoader'ы для каждого GraphQL-запроса
- Объявляйте DataLoader в resolve-методе, там где у вас возникает проблема N+1 Query
- Ничего страшного если для одного резолвера у вас создастся несколько DataLoader'ов (это когда резолвер используется на разных уровнях в GraphQL-запросе)

Пример запроса, когда один резолвер (связь Article -> Author) создаст несколько DataLoader'ов:

```graphql
{
  article(id: 5) {
    author { name } # DataLoader1
  }
  articles {
    title
    author { name email } # DataLoader2
  }
  lastComments {
    article {
      author  { name } # DataLoader3
    }
  }
}
```

**Чистого кода и меньше бесполезных запросов в ваш ~~дом~~ бэкенд!**

## Ссылки по теме

- [Код сервера без DataLoader](./server.js)
- [Код сервера c DataLoader](./dl-server.js)

# WPSN Knex In Depth

이 프로젝트는 WPSN OAuth 프로젝트에서 분기된 것입니다.

## SELECT

Knex는 SELECT 구문을 크게 신경쓰지 않고도 편하게 사용할 수 있도록 만들어져 있습니다. 하지만 테이블에 컬럼이 많은 경우, 특정 작업에서 필요한 컬럼만을 선택적으로 불러오면 웹 페이지의 응답 속도를 빠르게 할 수 있습니다.

```js
// 특정 사용자의 username와 avatar_url만 필요한 경우
knex('user')
  .select('username', 'avatar_url')
  .where('id', 1)
  .first()
  .then(data => {
    console.log(data)
  })
// { username: 'seungha', avatar_url: 'https://...' }
```

## JOIN

`JOIN` 구문을 사용하면 여러 테이블을 합쳐서 하나의 테이블로 만들 수 있습니다. Knex에서도 JOIN을 잘 지원하고 있습니다.

```js
// 사용자 이름과 게시글의 제목을 하나의 테이블로 출력하는 쿼리
knex('user')
  .select('user.username', 'article.title')
  .join('article', 'article.user_id', 'user.id')

// toString 메소드를 호출해보면
'select `user`.`username`, `article`.`title` from `user` inner join `article` on `article`.`user_id` = `user`.`id`'

// then 메소드를 호출해서 결과를 살펴보면
[ { username: 'user1', title: '제목1' },
  { username: 'user1', title: '제목2' },
  { username: 'user2', title: '제목3' } ]
```

다만 `join` 메소드를 사용할 때는 주의해야 할 점이 있습니다. Knex는 테이블의 각 행을 JavaScript 객체로 바꾸어서 저장하는데, 객체로는 이름이 같은 컬럼의 데이터를 구분할 수 있는 방법이 없습니다.

```js
> knex('user')
  .join('article', 'article.user_id', 'user.id')
  .first()
  .then(console.log)
// `id`라는 이름의 컬럼이 양쪽 테이블에 모두 존재하지만, 객체에는 하나의 id 컬럼만 나와있음
{ id: 1,
  provider: 'github',
  provider_user_id: '767106',
  access_token: '***************************************',
  avatar_url: 'https://avatars1.githubusercontent.com/u/767106?v=4',
  username: 'seungha-kim',
  user_id: 3,
  title: '제목1',
  content: 'lorem ipsum ...',
  created_at: 2017-10-16T13:07:49.000Z }
```

이 문제를 해결하기 위해, `select` 메소드와 SQL의 `AS` 구문을 사용할 수 있습니다.

```js
> knex('user')
  .select('user.id as user_id', 'article.id as article_id')
  .join('article', 'article.user_id', 'user.id')
  .first()
  .then(console.log)
```

## GROUP BY

Knex의 `groupBy` 메소드로 `GROUP BY` 구문이 포함된 쿼리를 작성할 수 있습니다.

```js
// 각 게시글에 달린 댓글의 갯수 세기
> knex('article')
  .join('comment', 'comment.article_id', 'article.id')
  .groupBy('article_id')
  .select('article_id', knex.raw('count(*) as comment_count'))
  .toString()
'select `article_id`, count(*) as comment_count from `article` inner join `comment` on `comment`.`article_id` = `article`.`id` group by `article_id`'
```

## 유연한 쿼리 빌더로서의 Knex

Knex 쿼리 빌더를 사용하면 유연한 방식으로 쿼리를 쌓아나갈 수 있습니다. **유연성**은 순수 SQL 혹은 ORM 대신에 쿼리 빌더를 사용하는 가장 큰 이유입니다.

### 느슨한 메소드 호출 순서

SQL의 엄격한 문법과 달리, Knex를 이용해서 쿼리를 작성하면 메소드의 호출 순서를 엄격하게 지킬 필요가 없습니다.

예를 들어, 아래 두 쿼리에 대해 `toString` 메소드를 호출하면 완전히 같은 결과가 나옵니다.

```js
getArticleById(id) {
  return knex('article')
    .join('user', 'user.id', 'article.user_id')
    .where('article.id', id)
    .first()
}
```

```js
getArticleById(id) {
  return knex('article')
    .first()
    .where('article.id', id)
    .join('user', 'user.id', 'article.user_id')
}
```

```js
// Node.js REPL
> query.getArticleById(1).toString()
'select * from `article` inner join `user` on `user`.`id` = `article`.`user_id` where `article`.`id` = 1 limit 1'
```

### 미리 작성해둔 쿼리를 이용해 다른 쿼리 만들기

Knex를 이용해 작성한 쿼리에 `then` 메소드를 호출한 적이 없다면, 그 쿼리는 아직 실제 데이터베이스와 통신이 이루어지지 않은 상태입니다. 이런 성질을 이용하면, 미리 작성해 둔 쿼리를 활용해 또다른 쿼리를 만들어내는 함수를 작성할 수 있습니다.

```js
// query.js
module.exports = {
  // ...
  getArticles() {
    return knex('article')
      .join('user', 'user.id', 'article.user_id')
      .orderBy('article.id', 'desc')
  },
  getArticlesWithCommentCount() {
    // 미리 작성해 놓은 `getArticles` 활용하기
    return this.getArticles()
      .select(knex.raw('COUNT(*) as comment_count'))
      .join('comment', 'comment.article_id', 'article.id')
      .groupBy('comment.article_id')
  },
  // ...
}
```

### 필요할 때 SELECT 하기

`select` 메소드는 여러 번 호출될 수 있습니다.

```js
knex('user')
  .select('username', 'avatar_url')
  .select('provider')
  .where('id', 1)
  .first()
  .then(data => {
    console.log(data)
  })
// { username: 'seungha', avatar_url: 'https://...', provider: 'github' }
```

이런 성질을 이용해, `query.js`와 같이 재사용을 위해 미리 쿼리를 작성한 쪽에서는 꼭 필요한 컬럼만을 선택하고, `bbs.js`와 같이 실제로 쿼리를 실행하는 쪽에서 추가로 필요한 컬럼을 선택하는 코드를 짤 수 있습니다. 이 기법을 통해 각 파일의 역할과 책임을 좀 더 세밀하게 분리시킬 수 있습니다.

```js
// query.js
getArticlesWithCommentCount() {
  return this.getArticles()
    .select(knex.raw('COUNT(*) as comment_count'))
    .join('comment', 'comment.article_id', 'article.id')
    .groupBy('comment.article_id')
}
```

```js
// bbs.js
router.get('/', (req, res) => {
  query.getArticlesWithCommentCount()
    // 추가로 필요한 컬럼을 select 메소드를 통해 선택한다.
    .select('article.id', 'article.title', 'article.content', 'article.created_at', 'user.username')
    .then(articles => {
      res.render('article/list.pug', {
        articles
      })
    })
})
```

---
emoji: 📓
title: Next.js에서 동적 sitemap 생성하기
date: '2023-08-22 23:00:00'
author: sunghaejoung
tags: 
categories: 블로그
---

## 0. Sitemap이란?

> 사이트맵(Sitemap)은 사이트에 있는 페이지, 동영상 및 기타 파일과 그 관계에 관한 정보를 제공하는 파일입니다. Google과 같은 검색엔진은 이 파일을 읽고 사이트를 더 효율적으로 크롤링합니다.[(링크)](https://developers.google.com/search/docs/crawling-indexing/sitemaps/overview?hl=ko)

<br/>
한마디로 사이트맵은 책의 목차라고 할 수 있다. 웹사이트에 방문한 검색엔진은 사이트맵을 통해 해당 웹사이트의 구조를 파악하고 누락될 수 있는 페이지까지 쉽게 크롤링할 수 있다. 결국 이 과정은 SEO에 긍정적인 영향을 미치게 된다.
리뉴얼되는 홈페이지에서는 회사가 보유하고 있는 매물을 홍보하는 목적성을 가지고 있었고, 각 매물의 상세페이지의 고객 유입을 높이기 위해서 사이트맵 생성은 필수적이였다.

## 1. 처음에 적용한 방식

처음에는 이 [블로그](https://medium.com/volla-live/next-js%EB%A5%BC-%EC%9C%84%ED%95%9C-sitemap-generator-%EB%A7%8C%EB%93%A4%EA%B8%B0-10fc917d307e)를 참고하여 코드를 작성했다. (해당 방식이 궁금하다면 링크를 통해 확인!)
이 방식은 우리 홈페이지와는 맞지 않는 부분이 있었다.

## 2. 특정 시점에만 사이트맵이 생성된다고?

해당 방식은 스크립트를 통해 특정 시점(참고한 블로그에서는 배포 시점)에서만 사이트맵이 업데이트 되었다.
하지만 매물은 실시간으로 새롭게 추가되고 기존에 있던 매물이 판매완료되기 때문에 사이트맵 생성 역시 실시간으로 이루어져야만 했다.

우리에게 맞는 방식을 생각해 봤을 때,
1. 매물이 업데이트 될 때마다 사이트맵을 생성하도록 스크립트 작성
2. 검색엔진이 sitemap.xml에 방문할 때마다 사이트맵 생성
1번 보다는 2번 방법이 좀 더 명확하고 간단하게 적용할 수 있을 것 같아서 2번 방법을 적용해보기로 했다!

## 3. 새로운 방식 도입 과정
### 1) /page/sitemap.xml 파일 추가
```js
import React from "react";

const Sitemap = () => {};

export default Sitemap;
```

검색엔진은 `/sitemap.xml`의 경로를 통해 사이트맵이 적힌 페이지에 방문하게 된다. 그리고 페이지 요청할때마다 사이트맵을 생성할 수 있도록 `getServerSideProps` [(공식문서 참고)](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props)에 사이트맵 생성 코드를 다음과 같이 작성했다.

```js
import React from "react";
const Sitemap = () => {};
export const getServerSideProps = ({ res }) => {
 const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
 <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
 <! - 여기에 사이트맵 URL이 추가됨 →
 </urlset>
 `;
res.setHeader("Content-Type", "text/xml");
 res.write(sitemap);
 res.end();
return {
 props: {},
 };
};
export default Sitemap;
```

### 2) 정적페이지 사이트맵 추가

globby를 사용해서 `page`디렉토리 안에 있는 모든 파일을 사이트맵으로 생성할 수 있게 코드를 작성했다. 그리고 Next.js에서만 사용되는 특수한 파일 (ex. `_app`, `_document`)과 동적 페이지를 포함한 사이트맵으로 생성하고 싶지 않은 파일은!을 사용해 제외시켰다.

```js
export const getServerSideProps = ({ res }) => {
  const baseUrl = {
    development: 'http://localhost:3000',
    production: 'https://example.com'
  }[process.env.NODE_ENV];
  
  const staticPages = (
    await globby([
      './src/pages/*.tsx',
      './src/pages/**/*.tsx',
      '!./src/pages/_app/*.tsx',
      '!./src/pages/_document/*.tsx',
      '!./src/pages/sitemap.xml.tsx',
      '!./src/pages/404/*.tsx',
      '!./src/pages/404/**/*.tsx',
      '!./src/pages/**/[path].tsx'
    ]) //!가 들어가면 해당 파일을 포함하지 않음 
  ).map(path =>
    path
      .replace('./src/pages/', '')
      .replace('.tsx', '')
      .replace(/\/index/g, '')
      .replace('index', '')
  );
  
  const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
      ${staticPages
        .map((url) => {
          return `
            <url>
              <loc>${baseUrl}/${path}</loc>
              <xhtml:link rel="alternate" href="${baseUrl}/${path}" />
              <lastmod>${format(new Date(), 'yyyy-MM-dd')}</lastmod>
              <changefreq>always</changefreq>
              <priority>1.0</priority>
            </url>
          `;
        })
        .join("")}
    </urlset>
  `;

// 생략
};
```

### 3) 동적페이지 사이트맵 추가
우선 동적페이지 생성에 필요한 데이터를 불러온다. 해당 프로젝트에서는 GraphQL을 사용해 데이터 통신을 하고 있어서 GraphQL 코드로 작성되었고 각자가 사용하는 통신방식을 사용하면 된다.
데이터를 호출한 이후에 앞서 작성했던 정적 페이지를 추가하는 코드 하단에 동일하게 작성해주면 끝..!!!

```js
// 데이터 호출
const { data } = await client.query({
  query: GET_FLEXES,
  context: { headers: { 'accept-language': 'en' } },
  fetchPolicy: 'network-only',
  variables: {
    filter: {
      exposed: true
    }
  }
});

const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
  <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
    <!-- 정적페이지 코드 -->
	  ${data?.flexes?.content
		.map(( { id }) => {
          return `
            <url>
              <loc>${baseUrl}/flex</loc>
              <xhtml:link rel="alternate" href="${baseUrl}/flex/${id}" />
              <lastmod>${format(new Date(updatedAt), 'yyyy-MM-dd')}</lastmod>
              <changefreq>always</changefreq>
              <priority>1.0</priority>
            </url>
          `;
        }).join('')}  
  </urlset>
`;
```

## 4.최종코드

```js
import React from "react";

const Sitemap = () => {};

export const getServerSideProps = ({ res }) => {
  const baseUrl = {
    development: 'http://localhost:3000',
    production: 'https://example.com'
  }[process.env.NODE_ENV];  
  
  const staticPages = (
    await globby([
      './src/pages/*.tsx',
      './src/pages/**/*.tsx',
      '!./src/pages/_app/*.tsx',
      '!./src/pages/_document/*.tsx',
      '!./src/pages/sitemap.xml.tsx',
      '!./src/pages/404/*.tsx',
      '!./src/pages/404/**/*.tsx',
      '!./src/pages/**/[path].tsx'
    ])
  ).map(path =>
    path
      .replace('./src/pages/', '')
      .replace('.tsx', '')
      .replace(/\/index/g, '')
      .replace('index', '')
  );  
  
  const { data } = await client.query({
    query: GET_FLEXES,
    context: { headers: { 'accept-language': 'en' } },
    fetchPolicy: 'network-only',
    variables: {
      filter: {
        exposed: true
      }
    }
  });
  
  const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
      ${staticPages
        .map((url) => {
          return `
            <url>
              <loc>${baseUrl}/${path}</loc>
              <xhtml:link rel="alternate" href="${baseUrl}/${path}" />
              <lastmod>${format(new Date(), 'yyyy-MM-dd')}</lastmod>
              <changefreq>always</changefreq>
              <priority>1.0</priority>
            </url>
          `;
        })
        .join("")}

	  ${data?.flexes?.content
		.map(( { id }) => {
          return `
            <url>
              <loc>${baseUrl}/flex</loc>
              <xhtml:link rel="alternate" href="${baseUrl}/flex/${id}" />
              <lastmod>${format(new Date(updatedAt), 'yyyy-MM-dd')}</lastmod>
              <changefreq>always</changefreq>
              <priority>1.0</priority>
            </url>
          `;
        }).join('')}  
    </urlset>
  `;

  res.setHeader("Content-Type", "text/xml");
  res.write(sitemap);
  res.end();

  return {
    props: {},
  };
};

export default Sitemap;
```

## 5. 정리
사이트맵에 대한 개념과 필요성 그리고 생성은 이번 프로젝트를 통해 처음 접하게 되었다. 처음 사이트맵 생성 코드를 작성할 때, 짧은 프로젝트 기간과 처음 적용하는 기술(Next.js와 GraphQL)에 대한 지식 부족으로 코드를 이해하면서 작성하지 못하고 그저 복사 붙여넣기,, 그리고 에러 해결만 반복할 뿐이였다.

이후 코드 하나하나 이해해가는 과정을 통해 page 디렉토리에 sitemap.xml을 생성하는 방법으로 대체할 수 있겠다 라고 생각했다. (이 모든 과정을 함께 해주신 명수님 감사드립니다)

이번 프로젝트를 통해 충분한 이해없이 작성한 코드를 하나씩 이해하며 깨닫는 과정의 즐거움을 느꼈고 앞으로의 프로젝트에서는 시간은 걸리겠지만 이해한 코드만을 작성할 수있도록 노력해봐야겠다

<br />

[참조 | How to Generate a Dynamic Sitemap with Next.js](https://cheatcode.co/tutorials/how-to-generate-a-dynamic-sitemap-with-next-js#generating-dynamic-data-for-our-sitemap)
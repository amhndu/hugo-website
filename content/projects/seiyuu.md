+++
title="SeiyuuList"
date="2020-03-22"
tags=["react", "seiyuu"]
description="A frontend for anilist that allows you to search anime voice actors and browse all of their character roles. Made in React. Unlike the pages from anilist/MAL, this gives you more information on the anime alongside the character, allowing you to sort the list however you want. Thus you won't get lost reading the super-long lists of popular seiyuus."
+++

A better seiyuu list.

It lets you search anime voice actors and browse all of their character roles. Unlike the pages from anilist/MAL, this gives you more information on the anime alongside their roles, this allows you to sort the list by say the anime popularity or scores.
This is particularly useful because popular voice actors tend to have many roles, so looking at their super long list can get you lost.


Try: https://seiyuu.now.sh/


Built with React/Material-UI/CRA and uses anilist's graphql API.


Source
----------
[GitHub](https://github.com/amhndu/seiyuu)


Run
------------
```
npm i
npm start
```

Deploy
------------
```
npm run build
```
Then host `/build/`  
Should also remove tracking code from layout.js if deployed on non-localhost


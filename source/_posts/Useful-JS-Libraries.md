---
title: 一些有用的 JS 第三方库
date: 2024-02-10 22:31:10
categories: 笔记
tags:
    - JavaScript
excerpt: 从课程 FullStackOpen 中截取的一小段关于各种“轮子”的介绍和推荐
---

{% notel blue fa-cirle-exclamation **引用** %}
本文来源于课程 [FullStackOpen](https://fullstackopen.com/), 原文地址： [Useful libraries and interesting links](https://fullstackopen.com/en/part7/class_components_miscellaneous#useful-libraries-and-interesting-links)
{% endnotel %}

# Useful libraries and interesting links

-   The JavaScript developer community has produced a large variety of useful libraries. If you are developing anything more substantial, it is worth it to check if existing solutions are already available. Below are listed some libraries recommended by trustworthy parties.

-   If your application has to handle complicated data, [lodash](https://www.npmjs.com/package/lodash), which we recommended in part 4, is a good library to use. If you prefer the functional programming style, you might consider using [ramda](https://ramdajs.com/).

-   If you are handling times and dates, [date-fns](https://github.com/date-fns/date-fns) offers good tools for that. If you have complex forms in your apps, have a look at whether [React Hook Form](https://react-hook-form.com/) would be a good fit. If your application displays graphs, there are multiple options to choose from. Both [recharts](https://react-hook-form.com/) and [highcharts](https://github.com/highcharts/highcharts-react) are well-recommended.

-   The [Immer](https://github.com/mweststrate/immer) provides immutable implementations of some data structures. The library could be of use when using Redux, since as we remember in part 6, reducers must be pure functions, meaning they must not modify the store's state but instead have to replace it with a new one when a change occurs.

-   [Redux-saga](https://redux-saga.js.org/) provides an alternative way to make asynchronous actions for [Redux Thunk](https://fullstackopen.com/en/part6/communicating_with_server_in_a_redux_application#asynchronous-actions-and-redux-thunk) familiar from part 6. Some embrace the hype and like it. I don't.

-   For single-page applications, the gathering of analytics data on the interaction between the users and the page is more [challenging](https://developers.google.com/analytics/devguides/collection/gtagjs/single-page-applications) than for traditional web applications where the entire page is loaded. The [React Google Analytics](https://github.com/react-ga/react-ga) library offers a solution.

-   You can take advantage of your React know-how when developing mobile applications using Facebook's extremely popular [React Native](https://facebook.github.io/react-native/) library, which is the topic of part 10 of the course.

-   When it comes to the tools used for the management and bundling of JavaScript projects, the community has been very fickle. Best practices have changed rapidly (the years are approximations, nobody remembers that far back in the past):

    -   2011 [Bower](https://www.npmjs.com/package/bower)
    -   2012 [Grunt](https://www.npmjs.com/package/grunt)
    -   2013-14 [Gulp](https://www.npmjs.com/package/gulp)
    -   2012-14 [Browserify](https://www.npmjs.com/package/browserify)
    -   2015- [Webpack](https://www.npmjs.com/package/webpack)

-   Hipsters seem to have lost their interest in tool development after webpack started to dominate the markets. A few years ago, [Parcel](https://parceljs.org/) started to make the rounds marketing itself as simple (which Webpack is not) and faster than Webpack. However, after a promising start, Parcel has not gathered any steam, and it's beginning to look like it will not be the end of Webpack. Recently, [esbuild](https://esbuild.github.io/) has been on a relatively high rise and is already seriously challenging Webpack.

-   The site [https://reactpatterns.com/](https://reactpatterns.com/) provides a concise list of best practices for React, some of which are already familiar from this course. Another similar list is [react bits](https://vasanthk.gitbooks.io/react-bits/).

-   [Reactiflux](https://www.reactiflux.com/) is a big chat community of React developers on Discord. It could be one possible place to get support after the course has concluded. For example, numerous libraries have their own channels.

---
layout: post
title:  "Reagent clientside routing with Bidi and Accountant"
---

# Reagent clientside routing with Bidi and Accountant

So, you are building a single page application with
[Reagent](https://reagent-project.github.io/) and want a clean client
side routing in place? Here's one way, inspired by
[Yogthos](https://github.com/yogthos) recipe of [a way to do it using
Secretary](http://yogthos.net/posts/2014-08-14-Routing-With-Secretary.html)
taking advantage of Reagent's `atom`.

We will be using [Bidi](https://github.com/juxt/bidi) for routing and
[Accountant](https://github.com/venantius/accountant) for hooking the
browser history (which in turn relies on HTML5 Pushstate). I am assuming
you have leiningen and figwheel working. I also assume you will use
those project links to check how to add Bidi and Accountant to your
project.

The outline of the approach is to let:

-   **Bidi do the routing translation.** It is designed to do this
    *bidirectionally*, hence the name. If you have a path, Bidi can
    match it to a route. If you have a route, Bidi can create the path
    that matches it.

-   **Accountant manage the browser location and history**, triggering
    the dispatching of routes by updating a `reagent/atom`. This relies
    on HTML5 **pushstate**. Check if [you can use
    it](http://caniuse.com/#search=pushstate) here.

-   **Reagent re-render** whatever parts of the UI that should change
    due to a route dispatch. We use a somewhat decorated `reagent/atom`
    called `session` from the
    [reagent-utils](https://github.com/reagent-project/reagent-utils)
    project to keep reagent informed.

I've created [an example
project](https://github.com/PEZ/reagent-bidi-accountant-example) showing
the recipe at work. Many of you will want to skip reading the rest of
this blog post and just study the example.

Setting up routes
=================

Bidi uses data structures to define the routing table. I am too new to
it to dare try to explain the grammar, check it up at the source. But at
least I can describe the following structure, from the routing table
used in the example project:

``` {.clojure}
(def app-routes
  ["/" {"" :index
        "section-a" {"" :section-a
                     ["/item-" :item-id] :a-item}
        "section-b" :section-b
        "missing-route" :missing-route
        true :four-o-four}])
```

As you can see the routing table is just data of regular clojure types.
You nest the path parts together, keeping things really DRY. Under `/`
we have four paths `empty` → `:index`, `section-a/` → two subpaths,
`section-b/` → `:section-b`, an (intentionally) missing route, and the
`true` path → `:four-o-four`, which catches everything not matching the
other paths.

The two subpaths of `section-a/` are: `empty` → `:section-a` and `item/`
→ `:section-a-item`. The latter path takes a parameter `:item-id` so
will match everything looking like `/section-a/item/anything` and will
put `anything` inside the matching routes `:route-params` as
`{:item-id anything}`. Bidi supports for checking the format of the
`anything`, but I leave that up to you to play around with.

This we have set up a site structure like so:

-   Home - `/`

    -   Section A `/section-a/`

        -   Item foo `/section-a/item/foo`

        -   Item bar `/section-a/item/bar`

        -   ...​

        -   Item baz `/section-a/item/baz`

    -   Section B \`/section-b/

The routing table and most of the code in the example project you will
find in
[`src/routing-example/core.cljs`](https://github.com/PEZ/reagent-bidi-accountant-example/blob/master/src/routing_example/core.cljs).

Defining pages
==============

The \"pages\" are reagent components. Duh. They are also multimethods,
and the routing will dispatch on a simple keyword:

``` {.clojure}
(defmulti page-contents identity)
```

The home page links to Section A and B. Note the use of `path-for` to
get the paths to use for the `<a>` tags:

``` {.clojure}
(defmethod page-contents :index []
  [:span
   [:h1 "Routing example: Index"]
   [:ul
    [:li [:a {:href (bidi/path-for app-routes :section-a) } "Section A"]]
    [:li [:a {:href (bidi/path-for app-routes :section-b) } "Section B"]]
    [:li [:a {:href (bidi/path-for app-routes :missing-route) } "Missing-route"]]
    [:li [:a {:href "/borken/link" } "Borken link"]]]])
```

The Section A component lists a few links to \"item\" pages. This
demonstrates how Bidi forms links for parameterized routes. Check the
`:section-a-item` entry in the routing table.

``` {.clojure}
(defmethod page-contents :section-a []
  [:span
   [:h1 "Routing example: Section A"]
   [:ul (map (fn [item-id]
               [:li {:key (str "item-" item-id)}
                [:a {:href (bidi/path-for app-routes :a-item :item-id item-id)} "Item: " item-id]])
             (range 1 6))]])
```

The Item component picks up the `:item-id` parameter from the
`:route-params` entry in the session atom. (More about this below.

``` {.clojure}
(defmethod page-contents :a-item []
  (let [routing-data (session/get :route)
        item (get-in routing-data [:route-params :item-id])]
    [:span
     [:h1 (str "Routing example: Section A, item " item)]
     [:p [:a {:href (bidi/path-for app-routes :section-a)} "Back to Section A"]]]))
```

Section B is really boring.

``` {.clojure}
(defmethod page-contents :section-b []
  [:span
   [:h1 "Routing example: Section B"]])
```

In the routing table we have a \"catch all\" route defined as
`[true :four-o-four]`, remember? This will get dispatched for anything
missing from the routing table. Typically, if you use the
bidirectionality of Bidi this will only happen when the user is trying a
mistyped link.

``` {.clojure}
(defmethod page-contents :four-o-four []
  "Non-existing routes go here"
  [:span
   [:h1 "404"]
   [:p "What you are looking for, "]
   [:p "I do not have."]])
```

We have also defined the route `:missing-route` to demonstrate that the
multimethod `:default` dispatching value can help with letting you know
when you have missimplemented (or missed to implement entirely) a method
for a *defined* route.

``` {.clojure}
(defmethod page-contents :default []
  "Configured routes, missing an implementation, go here"
  [:span
   [:h1 "404: My bad"]
   [:pre.verse
    "This page should be here,
but I never created it."]])
```

Ideally your CI pipeline can use this to catch this error before it hits
your users.

The page template component
===========================

All the pages in the silly app has the same header and footer, and then
embedd the active page component. Here the session atom gets derefed and
we pick out the route `:current-page` and then pick out the
corresponding component from the `page-components` map.

``` {.clojure}
(defn page []
  (fn []
    (let [page (:current-page (session/get :route))]
      [:div
       [:p [:a {:href (bidi/path-for app-routes :index) } "Go home"]]
       [:hr]
       (page-contents page) ;;
       [:hr]
       [:p "(Using "
        [:a {:href "https://reagent-project.github.io/"} "Reagent"] ", "
        [:a {:href "https://github.com/juxt/bidi"} "Bidi"] " & "
        [:a {:href "https://github.com/venantius/accountant"} "Accountant"]
        ")"]])))
```

-   This will \"dispatch\" a new \"page\" when the session reagent/atom
    changes.

Time to dispatch
================

In order to enable the dispatching we need to get the `page` component
into the browser's DOM:

``` {.clojure}
(defn on-js-reload []
  (reagent/render-component [page]
                            (. js/document (getElementById "app"))))
```

(Also, Figwheel is configured to call this function whenever it has
compiled new code.)

When the app initializes we configure the Accountant handlers. The
`:nav-handler` is responsible for updating the session atom whenever the
browser wants to visit a new path within the app. Here's is where we
meet the reciprocal friend of `bidi/path-for`; `bidi/match-route`. If
the matched route is parameterized the handler also inserts the
parameters into the session.

``` {.clojure}
(defn ^:export init! []
  (accountant/configure-navigation!
   {:nav-handler (fn
                   [path] ;; 1
                   (let [match (bidi/match-route app-routes path) ;; 2
                         current-page (:handler match) ;; 3
                         route-params (:route-params match)] ;; 4
                     (session/put! :route {:current-page current-page ;; 5
                                           :route-params route-params})))
    :path-exists? (fn [path]
                    (boolean (bidi/match-route app-routes path)))})
  (accountant/dispatch-current!) ;;
  (on-js-reload))
```

1.  Accountant grabs the path from the browser navigation, and ...​
2.  ...​ Bidi matches the route, using `bidi/match-route`.
3.  Derefed in the `[page]` component, triggering a \"dispatch\".
    `:route-params` to be de-refed by any component that needs them.
4.  Here is when the Reagent atom (`session/state` in this case) is
    updated triggering a re-render in the `page` component.
5.  This triggers the `:nav-handler`, updating the session so that the
    first `on-js-reload` call has something to work on.

That's it! Try the app and surf it. =)

However, during development you might find that it is a bother everytime
you have crashed it in a way that needs refreshing the browser window.
If you are not on the `:index` page you'll get Figwheel's 404 page.
That's why we need...​

Loading the app from \"outside\" the root path
==============================================

Before Accountant has taken responsibility for the browser navigation,
we need to replace Figwheels 404 page with our app's html template,
regardless which app path is loaded.

[claj](https://github.com/claj) showed me a really simple way yo solve
it. In the example project Figwheel is configured with a `:ring-handler`
set to `routing-example.server/handler`. The project's `:dev` profile is
configured to look for code in the `dev/` folder and there you will find
`server.clj` with the following contents:

``` {.clojure}
(ns routing-example.server)
(defn handler [request]
  {:status 200
   :body (slurp "resources/public/index.html")})
```

It doesn't even need explaining. However, **a few things can trip you up
if you try this**:

-   In the project file, the `:compiler` `:asset-path` entry for the
    `:dev` build use a relative path. Fix it so that it reads
    `:asset-path "/js/compiled/out"`

-   The template `index.html` also use relative paths for style sheets
    and script resources. Make those absolute and save yourself some of
    that precious hair on your head.

Room for improvement
====================

**Refreshing the browser resets the app.** Since refreshing means
reloading `index.html` we lose the benefit of Figwheel for these cases.

I don't know if the browser history can be hijacked in such a way so
that the app isn't jettisoned on refresh or when the user alters the url
in the location bar. But I know that I didn't have this problem when
using Secretary and hooking the goog.history events old-style using
paths prefixed with a `#`. So if someone could file a pull request for
Accountant supporting that we could configure things to use \#-prefixed
paths for development and (if the business allows) HTML5 pushstate for
productuon.

Feedback please, and PRs!
=========================

I am a clojure and clojurescript noob. Please help me on my journey with
PR's to the example project, feedback and tips on how to roll this in a
more idiomatic way.

Oh, and if you somehow missed where that [example
project](https://github.com/PEZ/reagent-bidi-accountant-example) is. =)

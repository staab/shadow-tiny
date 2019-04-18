Build with `npx shadow-cljs release library`

Get size with `ls -la build | awk '{ sum += $5; } END { print sum; }' "$@" -h build`. Currently this gives me `25133`, which is pretty much the size of cljs.core, trimmed down a bit by tree-shaking.

My chat log with @dnolen and @thheller explains why this is the case, and why it can't get much smaller:

jstaab [9:12 AM]
Hey there, I'm trying to optimize my bundle with shadow-cljs, in particular by trimming down cljs.core as much as I can. I've got a reproduction of just a small proof-of-concept here: https://github.com/staab/shadow-tiny

The problem I'm running into is that if I use anything in core, my bundle blows up in size; currently I'm just invoking a couple core functions, and the size of `cljs.core.js` alone is 97k. Is there any way to improve this? I was thinking of maybe using `refer-clojure` in all my namespaces, but the docs say "The only options for :refer-clojure are :exclude and :rename". I'm also not sure if this would work, because it does appear that google closure compiler is properly tree-shaking things (`cljs.core.js` is only 1.1k if I remove the body of my main function, and my other project's `cljs.core.js` is 136k). (edited) 

lilactown [9:22 AM]
have you used the shadow-cljs build report?
it can give you better insight into what’s taking up space

jstaab [9:23 AM]
I haven't on this project; I tried it on my main one yesterday and it crashed so I moved on
let me give it a quick go

lilactown [9:23 AM]
that might a good thing to report if you have the time

jstaab [9:23 AM]
This is my source code though:

```(ns shadow-tiny.core)

(defn ^:export main []
  (print (map inc [1 2])))
```

lilactown [9:24 AM]
I think what you’re going to run into is that the CLJS data structures weigh about 60-80kb un-gzipped

jstaab [9:25 AM]
Ok, so I guess there's just not really a good way to avoid that
Is there a way to use a CDN for cljs core?

lilactown [9:25 AM]
no, because it will tree-shake and remove things you don’t want
if you used a CDN for cljs.core it would be even bigger

jstaab [9:25 AM]
Bigger, but hopefully aggressively cached
Preact/redux are a total of 5k (gzipped), and I'd like to get cljs to compete with that somehow
it looks like my toy project is 88k gzipped
Is there any prior art on this you know of? I haven't been able to find any articles about tiny bundles online

lilactown [9:27 AM]
preact and redux are much smaller in scope than CLJS (the language)
it’s comparing apples to oranges

jstaab [9:28 AM]
It's true, but as far as delivering something to users, I'm more willing to suffer with js than bloat my bundles

lilactown [9:29 AM]
for perspective, immutable.js is ~64K minified

jstaab [9:29 AM]
ouch
Man, I love immutable data structures, but I'm not sure they're worth the cost

lilactown [9:30 AM]
I think they are :slightly_smiling_face:

jstaab [9:30 AM]
heh

lilactown [9:31 AM]
88K is honestly not enough that I would start to balk. that’s not going to strain my users

jstaab [9:31 AM]
hmm I think I'm misusing `du` too, it should be smaller than that (edited) 
`ls` gives me closer to 35k (edited) 

thheller [9:33 AM]
@jstaab CLJS isn't really suited for "tiny" bundles since you are always going to end up with `cljs.core` and the datastructures
its never going to compete with preact/redux (edited) 

jstaab [9:34 AM]
@thheller thanks for the confirmation, that's what I was beginning to suspect

thheller [9:34 AM]
but it'll easily beat preact/redux/immutable/lodash
which you are basically getting with `cljs.core`

jstaab [9:34 AM]
do you have any strategies for alleviating the burden? I'd love to use it _everywhere_, and with cljs.core aggressively cached on a CDN that seems feasible (edited) 

dnolen [9:36 AM]
there's really not much more you can do

valtteri [9:36 AM]
I’m setting up SPA routing without `/#`.  I’ve configured webserver to always return `index.html`. Basic stuff works but I’m having trouble with nested routes like `www.example.com/foo/bar`.  My build tries to find `/foo/js/compiled/out/goog/base.js` when it actually exists in `/js/compiled/out/goog/base.js`. Same for `deps.js` and `deps_cljs.js`. What knobs I need to turn to make those paths absolute?

thheller [9:36 AM]
use code splitting .. but no CDN caching cljs.core is not possible

dnolen [9:37 AM]
code splitting is the answer if you want to limit the initial payload - but fundamentally we start around 22K-25K gzipped and go up from there
given modern JS development - we are in fact very light weight

jstaab [9:37 AM]
CDN caching is not possible meaning not supported/worth supporting, or is there something more fundamental going on with how it's linked?

dnolen [9:37 AM]
and not really interested in investing more time beyond what Closure gives us

jstaab [9:37 AM]
Ha @dnolen I've been reading some Henrik Joreteg lately and it's inspiring me to change that

dnolen [9:37 AM]
sure
but I don't really sympathize that much with such efforts - Closure fundamentally does the only interesting thing that can be done
the only JS project that understands this is Rollup
hand optimizing payloads is a massive waste of time IMO

jstaab [9:40 AM]
I buy that for sure
Cool, thanks all for the answers, that was super helpful

dnolen [9:43 AM]
Svelte is also interesting - but I can't find any data on that on really large projects
but my hunch is once you factor in all the dependencies you actually need, internationalization etc.

jstaab [9:43 AM]
Svelte _is_ interesting

dnolen [9:43 AM]
it doesn't add up to much

jstaab [9:43 AM]
yeah, I trimmed down my moment-timezone bundle yesterday

dnolen [9:43 AM]
which is probably why don't find anyone bragging about non-toy stuff

jstaab [9:43 AM]
saved me 800k

dnolen [9:44 AM]
so suffice to say, I think ClojureScript and Closure are still state of the art, the real answer is make sure deps can be dead code eliminated
I think ClojureScript core is like 75,000-80,000 lines of generated JS or something (edited) 
and you get 22K gzipped as the starting point

jstaab [9:45 AM]
Yeah, that makes sense
If there were fewer macros, would bundle sizes shrink? I suppose run times would grow commensurately though

thheller [9:45 AM]
macros don't contribute to bundle size at all

dnolen [9:46 AM]
macros don't contribute

jstaab [9:46 AM]
Really? I thought they were evaluated during build time, so if I had `(defmacro x [] "blah blah blah")` and used it a bunch, wouldn't the output be bigger? Or do you mean it's just negligible?

thheller [9:47 AM]
the code they generate end up in the JS. the macros themselves don't. (edited) 
so yeah if you generate a bunch of code they can affect the bundle size if emit the same stuff over and over again

jstaab [9:49 AM]
I see what you mean, the output would be the same whether you used a macro or manually wrote everything out longform, unless your macro is bloating the output unnecessarily
Anyhoo, thanks for your time, both of you

dnolen [9:54 AM]
@jstaab right and if you don't use it, it will still get eliminated from the final build

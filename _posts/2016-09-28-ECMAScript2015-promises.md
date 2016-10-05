---
layout: post
title: Les promesses en ECMAScript 2015, ce qu'il faut savoir
tags: [javascript, ES6]
commentsIssueID: 1
---

# Les promesses en ECMAScript 2015, ce qu'il faut savoir

## Définition

> L'objet Promise (pour « promesse ») est utilisé pour réaliser des traitements de façon asynchrone. Une promesse représente une unique opération asynchrone qui n'a pas encore été complétée, mais qui est attendue dans le futur. 
>MDN

Plus concrètement, les promesses sont des objets apportant une nouvelle syntaxe au javascript qui permet de mettre à plat ce qu'on appelle techniquement le "code spaghetti ".


{% highlight js %}
// callback-based functions
function spaghettiCode() {
  asyncStuff(function (err, res) {
    if (err) { throw err; }
    moreAsyncStuff(function (err, res) {
      if (err) { throw err; }
      andMoreAsyncStuff(function (err, res) {
        if (err) { throw err; }
        oneLastAsyncStuff(function (err, res) {
          if (err) { throw err; }
        });
      });
    });
  });
}

// promise-based functions
function flatCode() {
  asyncStuff()
  .then(moreAsyncStuff)
  .then(andMoreAsyncStuff)
  .then(oneLastAsyncStuff)
  .catch(function (err) {
    throw err;
  });
}
{% endhighlight %}


>Les promesses en javascript sont définies par un standard nommé Promises/A+ (<https://promisesaplus.com/>).
Il existe plusieurs librairies qui l'implémentent, plus ou moins fidèlement, telles que Q, RSVP.js ou Bluebird. Je vais cependant uniquement parler dans cet article de l'implémentation native des promesses apportées par l'ES6.

## Promise 101

Commençons par quelques termes techniques. Une promesse peut avoir trois statuts différents : `pending`, `resolved` (ou `fullfilled`) et `rejected`. Il est possible d'instancier une promesse avec chacun de ces trois statuts.

Une promesse `pending` attendra sa résolution ou sa réjection.
Une promesse `resolved` appellera la fonction spécifiée avec la méthode `then`.
Une promesse `rejected` appellera la fonction spécifiée avec la méthode `catch`.

### Pending promise

{% highlight js %}
var promise = new Promise(function (resolve, reject) {});

promise.then(function () { 
  console.log('ok');
  // unreached code
});

promise.catch(function () {
  console.log('ko');
  // unreached code
});

// display nothing
{% endhighlight %}

### Pending promise about to be resolved

**Une `new Promise` est résolue lorsque `resolve` est appelé.**

{% highlight js %}
var promise = new Promise(function (resolve, reject) {
  resolve();
});

promise.then(function () { 
  console.log('ok');
});

promise.catch(function () {
  console.log('ko');
  // unreached code
});

// display:
// ok
{% endhighlight %}

### Pending promise about to be rejected

**Une `new Promise` est rejetée si `reject` est appelé, ou si `throw`est invoqué.**

{% highlight js %}
var promise = new Promise(function (resolve, reject) {
  reject();
});

promise.then(function () { 
  console.log('ok');
  // unreached code
});

promise.catch(function () {
  console.log('ko');
});

// display:
// ko
{% endhighlight %}

### Resolved promise

{% highlight js %}
var resolvedPromise = Promise.resolve();

promise.then(function () { 
  console.log('ok');
});

promise.catch(function () {
  console.log('ko');
  // unreached code
});

// display:
// ok
{% endhighlight %}

### Rejected promise

{% highlight js %}
var rejectedPromise = Promise.reject();

promise.then(function () { 
  console.log('ok');
  // unreached code
});

promise.catch(function () {
  console.log('ko');
});

// display:
// ko
{% endhighlight %}

## Promise wrapping

Les fonctionnalités natives de Javascript/Node.JS ne sont pas encore promise-based. Transformer des fonctions callback-based en fonctions promise-based est donc un bon entraînement et une étape primordiale dans le développement d'un projet promise-based.

{% highlight js %}
function setTimeoutPromise (duration) {
  return new Promise(function (resolve) {
    setTimeout(resolve, duration);
  });
}

// usage:
setTimeoutPromise(1000)
.then(function () {
  console.log('done !');
});
{% endhighlight %}

> En Node.JS, il existe des librairies qui se chargeront de convertir automatiquement des fonctions callback-based en fonctions promise-based, comme par exemple <https://github.com/digitaldesignlabs/es6-promisify>.

## Enchaînement de promesses

Il y a quelques points importants à connaître à propos de l'enchaînement des promesses.

`new Promise(function (resolve, reject) {})` retourne une promesse.

`promise.then(function () {})` retourne une nouvelle promesse qui sera :

**Résolue au retour de la fonction spécifiée en paramètre, ou si la fonction retourne une promesse résolue.**
**Rejetée si un `throw` est appelé dans la fonction, ou si celle-ci retourne une promesse rejetée.**

Lorsque la fonction spécifiée en paramètre de `then` retourne une promesse, sa résolution sera attendue avant la suite de l'enchaînement.

{% highlight js %}
var promise = Promise.resolve();

promise.then(function () {
  console.log(1);
})
.then(function () {
  return new Promise(function (resolve) {
    setTimeout(function () {
      console.log(2);
      resolve();
    }, 0);
  });
})
// wait for resolve to be called by setTimeout
.then(function () {
  console.log(3);
});

// display:
// 1
// 2
// 3
{% endhighlight %}

Dans l'exemple suivant, les promesses ne sont pas correctement chaînées, toutes les fonctions sont associées au `then`de la première promesse. Ces fonctions vont donc êtres appelées simultanément à la résolution de la promesse initiale.

{% highlight js %}
var simultaneousPromise = Promise.resolve();

simultaneousPromise.then(function () {
  console.log(1);
});

simultaneousPromise.then(function () {
  return new Promise(function (resolve) {
    setTimeout(function() {
      console.log(2);
      resolve();
    }, 0);
  });
});

simultaneousPromise.then(function () {
  console.log(3);
});

// enchaînement brisé, promesses simultanées
// 1
// 3
// 2
{% endhighlight %}

Dans l'exemple suivant, l'enchaînement est préservé grâce à la conservation de chaque nouvelle promesse.

{% highlight js %}
var chainedPromise = Promise.resolve();

chainedPromise = chainedPromise.then(function () {
  console.log(1);
});

chainedPromise = chainedPromise.then(function () {
  return new Promise(function (resolve) {
    setTimeout(function() {
      console.log(2);
      resolve();
    }, 0);
  });
});

chainedPromise = goodPromise.then(function () {
  console.log(3);
});

// enchaînement préservé
// 1
// 2
// 3
{% endhighlight %}

## Transmission de variable

Il est possible de transmettre une unique variable à travers une promesse.

{% highlight js %}
var promise = Promise.resolve(42, 41, 40);

promise.then(function () {
  console.log(arguments);
  // display: [42]

  return 'hello';
}).then(function () {
  console.log(arguments);
  // display: ['hello']

  return Promise.resolve({ foo: 'bar' });
}).then((function () {
 console.log(arguments);
 // display: [{ foo: 'bar' }]
});
{% endhighlight %}

## Error catching

La méthode `catch` permet d'intercepter les promesses rejetées.
A la suite d'un `catch`, l'enchaînement est préservé.

{% highlight js %}
var promise = Promise.resolve();

promise.then(function () {
  console.log('a');
  throw new Error();
  console.log('b');
})
.catch(function (err) {
  console.log('err');
});
.then(function () {
  console.log('c');
});
// display:
// a
// err
// c
{% endhighlight %}

Il est également possible de spécifier une fonction de `catch` en la plaçant en deuxième argument de `then` :

{% highlight js %}
var promise = Promise.resolve();

promise.then(function () {
  return Promise.reject(new Error());
})
.then(function thenFct() {
  console.log('ok');
}, function catchFct(err) {
  console.log('ko');
})
.then(function () {
  console.log('done');
});

// display:
// ko
// done
{% endhighlight %}

Cependant, cette syntaxe fait perdre votre projet en lisibilité.
[À lire sur ce sujet](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns#the-thensuccess-fail-anti-pattern).

Si plusieurs `catch` sont présents, seul le premier sera appelé.

{% highlight js %}
var promise = Promise.resolve();

promise.then(function () {
  return Promise.reject(new Error());
})
.then(function () {
  console.log('ok');
})
.catch(function (err) {
  console.log('ko1');
})
.then(function () {
  console.log('done');
})
.catch(function (err) {
  console.log('ko2');
});

// display:
// ko1
// done
{% endhighlight %}

Enfin, vous pouvez transmettre l'exception au `catch` suivant grâce à un `throw` ou à un une promesse rejetée.

{% highlight js %}
var promise = Promise.resolve();

promise.then(function () {
  return Promise.reject(new Error());
})
.then(function () {
  console.log('ok');
})
.catch(function (err) {
  console.log('ko1');
  throw err;
  // or
  return Promise.reject(err);
})
.then(function () {
  console.log('done');
})
.catch(function (err) {
  console.log('ko2');
});

// display:
// ko1
// ko2
{% endhighlight %}

Attention cependant, si vous invoquez `throw` dans une fonction asynchrone, l'erreur ne sera pas catchée !

{% highlight js %}
var promise = Promise.resolve();

promise.then(function () {
  return new Promise(function (resolve, reject) {
    setTimeout(function () {
      throw new Error();
    }, 0);
  });
})
.then(function () {
  console.log('done');
})
.catch(function (err) {
  console.log('ko');
});

// will kill the program
{% endhighlight %}

Il n'est pas obligatoire, mais recommandé, de throw un objet `Error`, car celui-ci embarque la stack d’exécution qui vous aidera à débugger. 

## promise.all

`promise.all` est une méthode prenant en paramètre un itérable (un Array par exemple) de promesses et renvoyant une promesse qui sera résolue lorsque toutes les promesses de l'itérable seront résolues.
Si l'une des promesses de l'itérable est rejetée, la promesse de `promise.all` le sera également.

{% highlight js %}
var p1 = Promise.resolve();
var p2 = new Promise(function (resolve) {
  console.log('a');
  resolve();
});
var p3 = new Promise(function (resolve) {
  setTimeout(function () {
    console.log('b');
    resolve();
  }, 0);
});

Promise.all([p1, p2, p3])
.then(function () {
  console.log('c');
});

// display:
// a
// b
// c
{% endhighlight %}

{% highlight js %}
var p1 = Promise.resolve();
var p2 = new Promise(function (resolve) {
  console.log('a');
  resolve();
});
var p3 = new Promise(function (resolve, reject) {
  setTimeout(function () {
    console.log('b');
    reject();
  }, 0);
});

Promise.all([p1, p2, p3])
.then(function () {
  console.log('c');
})
.catch(function () {
  console.log('ko');
});

// display:
// ko
{% endhighlight %}

## promise.race

`promise.race` est une méthode prenant en paramètre un itérable de promesses et renvoyant une promesse qui sera résolue ou rejetée dès lors qu'une des promesses de l'itérable sera résolue ou rejetée.

{% highlight js %}
var p1 = new Promise(function (resolve) {
  console.log('a');
  resolve();
});
var p2 = new Promise(function (resolve, reject) {
  setTimeout(function () {
    console.log('b');
    resolve();
  }, 0);
});

Promise.race([p1, p2])
.then(function () {
  console.log('c');
});

// display:
// a
// c
// b
{% endhighlight %}

{% highlight js %}
var p1 = new Promise(function (resolve, reject) {
  console.log('a');
  reject();
});
var p2 = new Promise(function (resolve) {
  setTimeout(function () {
    console.log('b');
    resolve();
  }, 0);
});

Promise.race([p1, p2])
.then(function () {
  console.log('c');
})
.catch(function () {
  console.log('ko');
});

// display:
// a
// ko
// b
{% endhighlight %}

Dans le cas suivant, p1 est résolue avant que p2 soit rejetée, la promesse finale est donc résolue car seul le premier résultat est pris en compte.

{% highlight js %}
var p1 = new Promise(function (resolve) {
  console.log('a');
  resolve();
});
var p2 = new Promise(function (resolve, reject) {
  setTimeout(function () {
    console.log('b');
    reject();
  }, 0);
});

Promise.race([p1, p2])
.then(function () {
  console.log('c');
})
.catch(function () {
  console.log('ko');
});

// display:
// a
// c
// b
{% endhighlight %}
---
layout: post
title:  "Datomic Component Entities"
date:   2016-01-02 19:59:38 +0100
categories: jekyll update
---

**Datomic - Component Entities**

Datomic has the concept of components entities where some 'inner' entities can be embedded in an 'outer' entity.
 
In the documentation these are demonstrated using order / order-lines.

Out of the box, Datomic provides features to manage lifecycles between the entity and the component entities:

- automatically link the order-lines inside the order
- track the parent order from any of the order lines (with reference to the _order)
- automatically retract all order-lines entities if and when the order is retracted

An important thing to observe is that order-line entities can be treated as any other entity in queries.

And you can access these features by passing around standard clojure map data structures.

Sometimes however there are use cases where the 'out of the box' functions don't match its needs. 

An example is a shopping cart. In these cases it is common to regularly have items added and deleted from the cart.

These aspects are not managed automatically. (link to the discussion at the end of the post)

The bare-bones schema for a cart is here:

{% highlight clojure %}
 {:db/id                 #db/id[:db.part/db]
  :db/ident              :cart/name
  :db/valueType          :db.type/string
  :db/cardinality        :db.cardinality/one
  :db.install/_attribute :db.part/db}
 {:db/id                 #db/id[:db.part/db]
  :db/ident              :cart/skus
  :db/isComponent        true
  :db/valueType          :db.type/ref
  :db/cardinality        :db.cardinality/many
  :db.install/_attribute :db.part/db}
{% endhighlight %}

Here is the most basic empty cart:

{% highlight clojure %}
(def cart {:cart/name "myCart"})
{% endhighlight %}

We can add this to Datomic with one additional property - a temporary DB/ID. Transactions are always in a list
 so we need to conjure up one of those.

{% highlight clojure %}
(defn ins1 [cart]
  (let [datomic-cart (conj [] (assoc cart :db/id (d/tempid :db.part/user)))]
    @(d/transact conn datomic-cart)))
{% endhighlight %}

The `d/tempid` function provides a placeholder structure for Datomic to replace during the transaction and
 looks like this `#db/id[:db.part/user -1000010]`

##Returning data rather than meta-data
In this case our function would return the transaction to the user. This is definitely OK in some cases but more often
 clients want the data back with the newly created ID.

When using Datomic this requires an extra step that is not needed with traditional databases. The creation of a new record
 (or any transaction for that matter) creates a new version of the database with the newly created data. But your connection
 refers to an earlier version of the database. 

To solve this we need two parts ... one is a new version of the database and another is a function to obtain the newly
 created IDs.

The transaction that is returned by the `d/transact` method provides several keys to help us out. Here is an example of
 a transaction that is returned from calling `ins1` as shown above:

{% highlight clojure %}
(ins1 cart)
=> {:db-before datomic.db.Db @c9055873 
    :db-after  datomic.db.Db @ba003976,
 :tx-data [#datom[13194139534313 50 #inst"2016-01-02T17:57:58.098-00:00" 13194139534313 true]
           #datom[17592186045418 63 "My-Cart" 13194139534313 true]],
 :tempids {-9223350046623220291 17592186045418}}
{% endhighlight %}

So we have the keys `db-after` and `tempids` that will give us what we need

{% highlight clojure %}
(defn save-new-cart [cart]
  (let [temp-id (d/tempid :db.part/user)
        tx-data (conj [] (assoc cart :db/id temp-id))
        tx @(d/transact conn tx-data)
        {:keys [db-after tempids]} tx
        cart-id (d/resolve-tempid db-after tempids temp-id)]
    (d/pull db-after '[*] cart-id)))
{% endhighlight %}

As an example:

{% highlight clojure %}
(save-new-cart conn cart)
=> {:db/id 17592186045420, :cart/name "My-Cart"}
{% endhighlight %}

This function returns the whole record which is what I needed but could just return the ID. There is no network cost
 to run the pull query and a direct lookup is highly efficient.
 
As an aside, we can see how we might generalise this to be used for other map data rather than just shopping carts:

{% highlight clojure %}
(defn save-new [thing]
  (let [temp-id (d/tempid :db.part/user)
        tx-data (conj [] (assoc thing :db/id temp-id))
        tx @(d/transact conn tx-data)
        {:keys [db-after tempids]} tx
        new-id (d/resolve-tempid db-after tempids temp-id)]
    (d/pull db-after '[*] new-id)))
{% endhighlight %}

Now we can move on from the background to the entity components themselves

##Entity components

You can see that the cart/skus are defined as a component. The schema for the skus:

{% highlight clojure %}
 {:db/id                 #db/id[:db.part/db]
  :db/ident              :sku/number
  :db/valueType          :db.type/long
  :db/cardinality        :db.cardinality/one
  :db.install/_attribute :db.part/db}
 {:db/id                 #db/id[:db.part/db]
  :db/ident              :sku/quantity
  :db/valueType          :db.type/long
  :db/cardinality        :db.cardinality/one
  :db.install/_attribute :db.part/db}
{% endhighlight %}

Here is an example cart with 2 skus:

{% highlight clojure %}
(def cart {:cart/name "myCart"
           :cart/skus [{:sku/number   12345
                        :sku/quantity 1}
                       {:sku/number   54321
                        :sku/quantity 2}]})
{% endhighlight %}

The nice thing is that we can just call `save-new` as defined above and it will still work. We don't need to add any 
 code to save carts which include items for purchase. Both of the skus will automatically be provided a distinct entity
 id and that will bubble back in the pull query. Like this:

{% highlight clojure %}
(save-new cart)
=>
{:db/id 17592186045422,
 :cart/name "myCart",
 :cart/skus [{:db/id 17592186045423, :sku/number 12345, :sku/quantity 1}
             {:db/id 17592186045424, :sku/number 54321, :sku/quantity 2}]}
{% endhighlight %}

 
Things become more complex when we want to perform updates / deletions (retractions in Datomic terms). 

First and most obvious we need to see if the cart has a `db/id` so we introduce higher level function to split out
 the two cases. If this feels imperative, I'm guilty as charged. Alternatives suggestions are welcomed. 

{% highlight clojure %}
(defn save-cart! [cart]
   (if (:db/id cart)
     (save-updated-cart cart)
     (save-new cart)))
{% endhighlight %}

So now let's see what we have to do with the update case...

{% highlight clojure %}
(defn calculate-updates [user-comps]
  (let [existing-comps (filter :db/id user-comps)
        additions (remove :db/id user-comps)
        datomicized-additions (map #(assoc % :db/id (d/tempid :db.part/user)) additions)]
    (concat existing-comps datomicized-additions)))
{% endhighlight %}

Firstly we add in the tempid, as we did for the cart, for any new components and return the combined list. Next we
 need to handle any retractions - items that are missing from the input data but that are present on the database.

{% highlight clojure %}
(defn calculate-retractions [list1 list2]
  (let [db-comps-set (into #{} (map :db/id list1))
        user-comps-sets (into #{} (map :db/id list2))
        diffs (clojure.set/difference db-comps-set user-comps-sets)]
    (map (fn [e] [:db.fn/retractEntity e]) diffs)))
{% endhighlight %}

Here we create a list of calls to `db.fn/retractEntity`. Finally we bring the two together into single list of
 transactions:

{% highlight clojure %}
(defn update [entity comp-key]
  (if-let [db-entity (d/pull db '[*] (:db/id entity))]
    (let [db-comps (comp-key db-entity)
          user-comps (comp-key entity)
          updated-entity (assoc entity comp-key (calculate-updates user-comps))
          retractions (calculate-retractions db-comps user-comps)
          ]
      (if (empty? retractions)
        (vector updated-entity)
        (conj retractions updated-entity)))))
{% endhighlight %}

##Database or user space function?
The above code is shown as a set of functions but in the end I opted to install the code as a database function. The
 main reason for this choice was to ensure atomic operation. If one is comparing database structures and userland
 structures there are race conditions that are avoided completely when using database functions.

This is the method defined as a database function called `component-crud`

{% highlight clojure %}
(def component-crud
  (d/function
    '{:lang   "clojure"
      :params [db entity comp-key]
      :code   (if-let [db-entity (d/pull db '[*] (:db/id entity))]
                (let [db-comps (comp-key db-entity)
                      user-comps (comp-key entity)
                      db-comps-set (into #{} (map :db/id db-comps))
                      user-comps-sets (into #{} (map :db/id user-comps))
                      diffs (clojure.set/difference db-comps-set user-comps-sets)
                      retractions (into [] (map (fn [e] [:db.fn/retractEntity e]) diffs))
                      existing-comps (filter :db/id user-comps)
                      additions (remove :db/id user-comps)
                      datomicized-additions (map #(assoc % :db/id (d/tempid :db.part/user)) additions)
                      comp-entities (into [] (reduce into [existing-comps datomicized-additions]))
                      updated-entity (assoc entity comp-key comp-entities)]
                  (if (empty? retractions)
                    (vector updated-entity)
                    (conj retractions updated-entity))))}))
{% endhighlight %}

To install this in the database, we need to transact in the definition

{% highlight clojure %}
(defn install-crud-fn [conn]
  @(d/transact conn [{:db/id #db/id [:db.part/user]
                      :db/ident :component/crud
                      :db/doc "Handle CRUD for component entities"
                      :db/fn component-crud}]))
{% endhighlight %}

To invoke the `component-crud` database function

{% highlight clojure %}
(defn- save-updated-cart [conn cart]
  "Update: embedded skus will be handled by the DB CRUD function"
  (let [tx-data [[:component/crud cart :cart/skus]]
        tx @(d/transact conn tx-data)
        db-after (:db-after tx)]
    (d/pull db-after '[*] (:db/id cart))))
{% endhighlight %}

So its definitely a win to have atomicity for certain functions. However database functions are more awkward to
 implement and invoke than standard clojure functions. They also have fewer affordances than standard functions:
 
 - they must be a single expression rather than a few smaller functions
 - diagnosing and debugging issues is not so easy if there are problems with the data at execution time

On the upside, it is still all standard Clojure so it's a huge win from an expressivity and eco-system perspective
 compared to DB functions in other embedded language systems in major databases such as Oracle, Postgres or MySQL.

##Should there be more out of the box from Datomic?

Yes and no... its hard to argue that these needs are universal. In fact, the semantics of retraction are not obvious
 or easily agreed. For example, should elements not in the list be automatically deleted? In this case yes, but we
 can imagine many conditions where that would not be the most obvious / desired behaviour.
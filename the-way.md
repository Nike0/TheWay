# The Way

It's no secret that at Uscreen we follow a pretty specific opinionated vision. We have chosen it based both on our own expertise and the experience of other big companies such as Basecamp and Shopify.

The goal of this document is to help you understand ours paradigms and approaches.

This Is The Way.

## Domain Driven Rails Way

This is how we'd name it. Ideologically we support [Rails Doctrine](https://rubyonrails.org/doctrine) and trying to find a good mix of following [Rails Standard Practices](https://guides.rubyonrails.org/) and building Business Logic Language-driven models.

Let's dive into the components:

### Rails Way

* The Rails-way to approach domain model is the `model` folder. This is the main focus point.
* The glue between it and the views would be a `controller`.
* A controller would render json data to views by using `serializers`
* External API wrappers and gem-like infrastructural things will go into `lib`.

You get the idea:

* Basic set of Rails folders, same as when you first learned Rails
* No additional patterns — we don't use services, we banned form objects, we won't add repositories and entities.
* We don't use dry-rb and don't deviate too much from the opinionated Rails way. Yes, we might use RSpec and Sidekiq instead of Minitest and Rescue, but the core web app structure remains as pure Rails as possible.

### Domain Driven

We invest 99% of our thinking time into tuning our Domain Model.

We define the term `model` as a shorter synonym for Domain Model. All business logic lives in the `model` folder.

**Note: `model` folder is not only for `ActiveRecord`, but also for `ActiveModel::Model` and just for plain classes. This is the folder for all your business logic. Persistence is just an optional detail of some of the models.**

(Examples will follow)

**When building our models, we optimize the cost of translating business logic language into the code:**

* Nouns of your business logic language should become classes
* Verbs that you naturaly use when describing a feature — should become method names (E.g. not `call!` 😂)
* Your classes should have the same relationships as in real life
* You should balance the weight ("fat/skinny") of your models — both might be bad
* We try to avoid words which are part of implementation and not business logic.

💡 Try to think as close to business terms as possible and try to minimize the amount of "technical glue" and technical layers (ideally zero)

**When adding business logic, we link everything to existing models:**

* Any new business logic is directly attached to existing models.
* When describing a usecase, we figure out which Nouns perform which Actions (verbs). We map this to classes, relationships and methods.
* When a user story becomes so complex that we get either "fat controller" or "fat model" — we extend our domain language so that the story becomes shorter with new Nouns/Verbs.
* When we need a utility namespace, we use the model representing the main subject of that utility.

Classes in the `model` folder represent the Nouns in the domain language. Public methods of these classes represent Verbs. Relationships are represented by either AR associations (if model is persisted) or via methods/delegate (e.g. `store.transactional_fees`)

**We fully utilize AM/AR functionality:**

* We use callbacks extensively when in your **business logic language** the case sounds like **every time X happens we do Y**.
* We don't use callbacks to describe specific usecases. Avoid `after_create if: created_from_api_controller` -> put this logic just in `api_controller`
* We use virtual attributes to describe specific reusable business usecases (which don't fit controller). Example: `password_confirmation` field for "User must confirm the password twice"-case.
* We use validations on the model instead of creating any additional patterns like form objects.

## Details and examples

### Nouns and verbs should become classes and methods

**Rationale:**

* Cheaper translation of natural language words into classes
* Less chance of mistakes: bugs/features/changes are described by using exact words as your classes
* Forces you to invest more resources into thinking about Business Language and ask correct questions: "In which model do I put this?", "Hmmm, this looks like it doesn't fit into any existing model! => Time to introduce a new Noun to the language", etc.

🟥 Bad:

```
# services/stores/transactional_fees_calculator.rb

module Stores
  class TransactionalFeesCalculator
    ...

    def initialize(store)
      @store = store
    end

    def apply_fees!(fees)
      @store.setting.update!(convert_fees_to_usd(fees))
    end

    private

    def convert_fees_to_usd
      ...
    end
  end
  ...
end

# services/stores/transactional_fees_reporter.rb

module Stores
  class TransactionalFeesReporter
    ...

    def initialize(store)
      @store = store
    end

    def call
      API::Reporter.report(@store.setting.fees.to_json)
    end

    private

    def convert_fees_to_usd
      ...
    end
  end
  ...
end

# Usage:
Stores::TransactionalFeesCalculator.new(store).apply_fees!(new_fees)
Stores::TransactionalFeesReporter.new(store).call
```

* 🔴 You don't say "We use a fees calculator for this store to apply fees" in normal human language. You'd say that "Apply fees for that store" without any "calculator" word, it's an extra term adding **more mental complexity and more cost to translate** from/to business logic language.
* 🔴 You have a term `Fees` used in all business calls, feature requests, chats, support requests, etc — but this business logic term is harder to discover from code directly, because there is no such model.
* 🔴 It is in `services` folder == you jump back and forth from model
* 🔴 The division is too specific and too narrow. It's impractical and forces a lot of jumps + obscures natural discoverability. Just imagine if in ActiveRecord instead of `user.save!`, `user.create!`, `user.delete` you had `ActiveRecord::SaveService.new(user).call`, `ActiveRecord::CreateService.new(user).call`, `ActiveRecord::DeleteService.new(user).call` used in whole project, just imagine this clumsy horror :)
* 🔴 The presence of the `call` method implies you are using a class as a method. Because normally you "call methods" and "create classes", it gets strange when you start to "call classes". We have an internal joke that we subtract 100$ from salary for each `call` method and send to charity 😂
* 🔴 This is practically directly against the [Exact Beautiful Code](https://rubyonrails.org/doctrine/#beautiful-code) Doctrine Principle: it is long, clumsy, doesn't sound like DSL. Also check the last example `people.include? person` vs `person.in? people` in the [Exact Beautiful Code](https://rubyonrails.org/doctrine/#beautiful-code) Doctrine Principle — we should care about what's the subject, what comes first in our "sentences" of code.

🟩 Good:

```
# models/stores/transactional_fees.rb

module Stores
  class TransactionalFees
    def initialize(store)
      @store = store
    end

    def apply!(fees)
      @store.setting.update!(convert_fees_to_usd(fees))
    end

    def send_report!
      API::Reporter.report(@store.setting.fees.to_json)
    end

    private

    def convert_fees_to_usd
      ...
    end
  end
  ...
end

class Store < ApplicationRecord
  def transactional_fees
    @transactional_fees ||= Stores::TransactionalFees.new(self)
  end
end

# Usage:
store.transactional_fees.apply!(new_fees)
store.transactional_fees.send_report!
```

* 🟢 Translation cost is low. You use same nouns and verbs as in business language. "Store" has "Transactional fees", which you either apply or send. You'd use the same phrase even out of programming context.
* 🟢 You add a noun reflecting a business language term. Now you have an entry/grouping point for everything transactional-fees-related. Better discoverability.
* 🟢 No jumps back and forth, everything in one file which is balanced: not too slim and not too fat.
* 🟢 You "call" methods, you "create" classes.
* 🟢 Noun is class name. Action/Verb is method name 👍
* 🟢 This better follows [Exact Beautiful Code](https://rubyonrails.org/doctrine/#beautiful-code). It is shorter, more natural sounding, contains less "glue-like" constructions.

### We try to avoid words which are part of implementation and not business logic

Implementation details don't need a part in the object name. They speak for themselves: you don't need a word "Creator" in the name — if it has a `create` inside — it's clear that it's a creator.

💡 If your noun ends in a "ER", "OR", especially "Manager", "Builder", "Creator" or "Calculator" — you probably missed a better term.

*Exceptions to that rule: TwittER is OK as it's a brand already; ToastER is also OK as it's a known real life object. You get the idea.*

**Cheatsheet:**

* If you wanna put a `Builder` or `Creator` word at the end, like `UserCreator` — then you want a static `def self.create_from...` method on `User`
* If you wanna put a `Calculator` word at the end — then figure out what it calculates. This will be your model where you'll just have `def calculate`
* `Manager` is a buzzword which means nothing :)

Also, as you must tune your Nouns (classes) and Verbs (methods) to be able to form human-readable sentences from them — appearance of any "Do-er" pattern might mean just bad division overall.

🟥 Bad:

```
# models/stores/transactional_fees_calculator.rb

module Stores
  class TransactionalFeesCalculator
    ...
  end
  ...
end

# models/stores/transactional_fees_reporter.rb

module Stores
  class TransactionalFeesReporter
    ...
  end
  ...
end

# Usage:
store.transactional_fees_calculator.calculate
store.transactional_fees_reporter.generate
```

🟩 Good:

```
# models/stores/transactional_fees.rb

module Stores
  class TransactionalFee
    def initialize(store)
      @store = store
    end

    def calculate
      ...
    end

    def generate_report
      ...
    end
  end
  ...
end

# Usage:
store.transactional_fees.calculate
store.transactional_fees.generate_report
```

🟩 Equally good (If your report is so big that it deserves its own class):

```
store.transactional_fees.calculate
store.transactional_fees.report.generate
```

You must find Nouns which would represent a known term and add actions to them.

* 🔴 Do not create objects which do a single action
* 🟢 Create objects which represent a single concept

### Your classes should have same relationships as in real life

**Rationale:**

* Cheaper translation of business logic terms into code
* Less chance of mistakes: bugs/features/changes are described by using exact words as your classes
* Forces you to invest more resources into tuning our Domain Model and its hierarchy == exactly what we want to invest our time into

🟥 Bad:

```
class Stream < ApplicationRecord
  belongs_to :content
end
```

* 🔴 Expensive translation from business language to code. Stream is a type of content and here you won't easily discover that there is a separate entity somewhere, which for some reason "belongs to" a content.
* 🔴 You get all disadvantages of a failed "inheritance" OOP design choice, we won't cover these here.

🟩 Good:

```
class LiveEvent < Content
  default_scope -> { live_event }
end
```

* 🟢 Translation cost is low.
* 🟢 Better grouping/encapsulation and other benefits of a well-designed inheritance structure.

### You should balance the weight of your models

Both too fat/too skinny models/controllers might indicate flaws in your term grouping and you might need to add more models.

*Example: when tired of saying "a home appliance consisting of a thermally insulated compartment and a heat pump that blahblah", we add a term named "Fridge", define its relationship with other terms, define verbs applicable on that noun.*

What is important in this example is that we are **not** adding any single-function-objects: no "door opener", "door closer", "food preserver", "compartment-insulator". Just Fridge. Which contains not a single function, but many functions by design.

When your `Fridge` becomes too "fat", because you become more interested in mechanics of its heat pump and add 12 methods related to it — our way is introducing a `HeatPump` model, defining its relationship with the Fridge (`has_one`), delegating some of the `Fridge` functions.

Given you already do this in natural language and in business language as well, usually most of the work is already done and you need to map these two as close as possible.

🟥 Bad:

```
models
├── user.rb  # 80 methods, 1000 lines of code
├── store.rb # 60 methods, 600 lines of code
└── video.rb # 60 methods, 5000 lines of code
```

🟥 Also bad:

```
models
├── user.rb  # 10 methods, 100 lines of code
├── users
      ├── twitter_poster.rb # 1 method, 10 lines of code
      ├── twitter_tweet_creator.rb # 1 method, 10 lines of code
      ├── twitter_profile.rb # 1 method, 10 lines of code
      ├── twitter_sync.rb # 1 method, 10 lines of code
      ├── profile_personal.rb # 1 method, 10 lines of code
      ├── profile_updates.rb # 1 method, 10 lines of code
      ├── profile_private.rb # 1 method, 10 lines of code
      ...
```

* 🔴 Too many jumps to get the whole idea.

🟢 Good:

```
models
├── user.rb  # 10 methods, 100 lines of code
├── users
      ├── twitter.rb # 10 methods, 100 lines of code
      ├── analytics.rb # 10 methods, 100 lines of code
      ├── profile.rb # 10 methods, 100 lines of code
      ├── order.rb # 10 methods, 100 lines of code
      ├── resume_watching.rb # 10 methods, 100 lines of code
      ...
├── store.rb # 10 methods, 100 lines of code
├── stores
      ├── transactional_fee.rb # 10 methods, 100 lines of code
      ├── analytics.rb # 10 methods, 100 lines of code
      ├── collected_fee.rb # 10 methods, 100 lines of code
      ├── mrr_graph.rb # 10 methods, 100 lines of code
      ...
└── video.rb # 10 methods, 100 lines of code
├── videos
      ├── analytics.rb # 10 methods, 100 lines of code
      ...
```

* 🟢 Encapsulated
* 🟢 Single responsibility (❗️ Do not confuse with "Single Function", which is more an anti-pattern in our approach)
* 🟢 Testable
* 🟢 Clean/readable, short, DRY
* 🟢 Related functionality is close, you don't jump too much
* 🟢 Every benefit you could get with services without [their downsides](https://intersect.whitefusion.io/the-art-of-code/why-service-objects-are-an-anti-pattern)

### Any new business logic is directly attached to existing models

The idea here is that we grow our graph of direct dependencies between entities of our business logic. A Person *has friends*, a person *lives in* a City, a person *has a twitter*, etc.

Have you ever heard of Mind Maps? We strive to have same kind of structure of our `model` folder.

Let's imagine we had a model called `Health`, so:

```
class Health < ActiveRecord
  belongs_to :person

  def overall_summary
    ...
  end

  def weak?
    ...
  end

  def good?
    ...
  end

  ...
end
```

, then an ideal example of `model` folder planning would be practically an image I found when googling "Mind map":

![image](https://user-images.githubusercontent.com/3010927/117964801-ba44c700-b32a-11eb-938d-bf6640d73263.png)

Given that "Health" is something a "Person" has, this would mean the following structure of our model folder:

```
models
├── person.rb
└── persons
      ├── health.rb
      └── healths
            ├── diet.rb
            └── diets
                  ├── fruit.rb
                  ├── vegetable.rb
                  └── vegetables
                        ├── vitamins.rb
                        ...
```

*Note: there might be an exception to this, when you really introduce a new meaningful term to the business logic language, so you have a brand new model on top level.*

*Note 2: also, if you need a persistent record, you'd usually create a top-level class and inherit from ActiveRecord.*

🟥 Bad:

```
models
└── person.rb # 100 methods, 1000 lines of code, including twitter, profile and whatnot.
```

🟥 Even worse:

```
models
└── person.rb

services
├── application_service.rb
└── twitter_manager
      ├── profile_follower.rb
      └── tweet_creator.rb
```

🟢 Good:

```
models
├── person.rb
└── persons
      ├── twitter.rb
      └── twitters
            └── profile.rb
            ...
```

See the resemblance of this approach with the mind map of `Health` above.

One more example:

🟥 Bad:

```
models
└── person.rb

services
├── application_service.rb
└── twitter_manager
      ├── profile_follower.rb
      └── tweet_creator.rb

# models/person.rb
class Person < ApplicationRecord
  # nothing about twitter
end

# services/twitter_manager/tweet_creator.rb
module TwitterManager
  class TweetCreator < ApplicationService
    def call
      ...
      TwitterAPI::Tweet.post(...) # TwitterAPI belongs to `lib` folder, not `model`
      ...
    end
  end
end
# services/twitter_manager/profile_follower.rb
module TwitterManager
  class ProfileFollower < ApplicationService
    def call
      ...
    end
  end
end

# Usage
TwitterManager::TweetCreator.new(person, text: 'hello').call
TwitterManager::ProfileFollower.new(person, profile).call
```

* 🔴 Expensive translation from business language to code
* 🔴 More entities than there are actual nouns in natural language
* 🔴 Relationship between entities is not the same as relationship between corresponding business terms
* 🔴 The "single function"-constraint forces all these classes to be too slim == too many jumps to "grasp the whole picture", discovering "common" logic between them will be harder, etc.
* 🔴 Hard to discover the business links. In reality, "John has a Twitter, writes to it and follows others profiles" — this is how the things work. This simple and obvious real life sentence is hard to discover: `person.rb` doesn't mention twitter at all in any way. You need to use either static analysis tools or draw additional usage diagrams to understand this. And both things mean your code starts to be so complex, that you can't read it without "maps" and tooling.

🟢 Good:

```
models
├── person.rb
└── persons
      └── twitter.rb

# models/person.rb
class Person < ApplicationRecord
  def twitter  # This is basically just our way of saying "has_one :twitter", but when twitter is not AR/AM.
    @twitter ||= Persons::Twitter.new(self)
  end
end

# models/persons/twitter.rb
module Persons
  class Twitter
    def tweet(text)
      ...
      TwitterAPI::Tweet.post(...) # TwitterAPI belongs to `lib` folder, not `model`
      ...
    end

    def follow_profile(profile)
      ...
    end
  end
end

# Usage
person.twitter.tweet('hello')
person.twitter.follow_profile(profile)
```

* 🟢 Cheap translation from business language to code. In real language, "John has a Twitter, tweets there and follows others profiles", in code: `john.twitter.tweet`, `john.twitter.profile`. It is as close as it can get, minimal translation cost.
* 🟢 Same amount of Nouns in business logic and in code. "**person** has **Twitter**, tweets there and follows others **Profiles**". We have `class Person`, `class Twitter` and might have a `Profile` model later.
* 🟢 Relationship between entities is exactly the same as in reality, same dependency graph.
* 🟢 On one hand, everything is in the same file/place, nearby, easy to see everything at once and grasp the whole picture.
* 🟢 On the other hand, each model maintains the not too slim/not too fat balance.

### When describing a use case, we figure out which Nouns perform which Actions

We do not create classes for scenarios / business cases / stories themselves. Such approach would not improve our interconnected graph of all business entities.

When we have a story, let's say, "When a user registers via sign-up page, we sync their secret data to SuperExternalIntegration", we don't create any object which means the story itself. No `UserRegistrationViaSignUp` (with or without "service" word)

Instead, we decompose that story into its parts:

* The **when** part goes into controller or rake task in most cases. `via sign-up page` practically translates into a very specific controller#action
* We figure out the acting **nouns**/**terms** which play roles in the scenario: `User`, `SuperExternalIntegration` (`secret data` word is just a glue)
* We figure out the main subject, the actor who plays the main role in this ~movie~ story: `User`, obviously
* We map and rephrase the story in the following way: `When POST to /api/blahblah happens, User is synced to SuperExternalIntegration`

Then we add

```
model
├── user.rb
└── users
     └── super_external_integration.rb
```

and

```
class User < ActiveRecord
  delegate :sync_secret_data, to: :super_external_integration

  def super_external_integration
    @super_external_integration ||= SuperExternalIntegration.new(self)
  end
end
```

Because the "when user registers" in reality means "When POST to /api/blahblah happens", we go to:

```
class BlahBlahController < ApplicationController
  def create
    ...
    user = User.new(form_params)
    if user.save
      user.sync_secret_data
    else
    ...
  end
end
```

And that's it.

#### The 'When' part

‼️ ‼️ ‼️ **Super important: please pay critical attention to the "When" part of your user story.**

Let's imagine that in the example above instead of

* When a user registers via sign-up page, we sync their secret data to SuperExternalIntegration

we had:

* When a user registers anywhere in our application, we sync their secret data to SuperExternalIntegration

**The difference between "via sign-up page" and "in our application" is critical** and drastically changes the taken approach, because:

* "When a user registers via sign-up page" == translates to "When POST to /api/blahblah happens"
* "When a user registers anywhere in our application" == translates to "after_commit on User model"

And while `POST to /api/blahblah` maps to a **controller**, `Anywhere` maps to a **model callback** on a real or a virtual model attribute.

This is where you must distinguish **common logic** vs **specific case logic**. Both types or errors are costly and very painful:

* If you put common logic into a specific controller — then you repeat it in many controllers (because it's common, you'll have to repeat). Then you change 1 place and forget to change the other places.
* If you put specific logic into a model callback — then your model callback will start having million conditions inside it and you'll get all the problems, usually associated with callbacks.

💡 **Good way to figure out:** ask yourself the question "If the same logic appears in 2-3 more places, will these places **change together?**"

* If the places change together always (e.g. when creating a store — create a store owner; by design: no store can be without owner) — pick a callback
* If the places just have same code, but are not actually related (e.g. 2-3 APIs just accidentally return same format; But that's not guaranteed at all in future) — it's better not to create a general abstraction for them — put in these controllers.

💡 You will also have **middle** situations in between **common** and **specific**. This is where you'd need to apply your expertise and where things might become a bit subjective, but in these cases you are encouraged to use what Rails provides out of the box:

* scoped validations. e.g. `validate :blahblah, on: :lead_capture`
* virtual attributes. e.g. `attribute :terms_conditions` and `validate :terms_conditions, blahblah, if: ->(user) { user.admin? and user.store.blahblah }`
* additional models, e.g. if we have a model `User` and have only specific validations/checks/logic for `client` role, then we can add an `EndUser` model with `default_scope { client }` and put it there

### When we need a utility namespace, we use the model

Use `def self.` static methods on Models.

🟥 Bad:

```
class User
  ...
end

class UserCsvCreator
  def create_from_csv
    csv = ... # load and parse csv
    User.create(csv)
  end
end
```

* 🔴 Expensive translation from business language to code. We definitely won't use such noun as "UserCsvCreator" in our language.
* 🔴 More entities than there are actual nouns in natural language
* 🔴 Fact of being creatable from CSV is less discoverable

🟢 Good:

```
class User
  def self.create_from_csv
    csv = ... # load and parse csv
    User.create(csv)
  end
  ...
end
```

* 🟢 Cheap translation from business logic language to code
* 🟢 Less jumps, easier to discover

## Q&A

**Q: So you write that we should map to real business logic terms. But what if we need a purely technical class to represent a row in the database, because this storage way would be optimal and cheaper, while in real life language you won't have a similar entity. What do we do?**

A: In this case obviously you'd have one more model for the "technical purpose". But we still try to give it a meaning, like `FeesMonthlySummary` to indicate the closest "real life meaning" of such an entity. And if the entity is 100% technical, like Frontier, then you usually just put it into the `lib` folder

**Q: In your mind-map example you draw a very deep tree and I see `fruit.rb` end up in the `models/health`, while I think they should exist on their own. What do you mean?**

A: In this example the file `models/healths/diets/fruit.rb` is inside `diets` folder, so it means more "Fruit-part of the diet". There might be a top-level `models/fruit.rb` separate from that. And there might be multiple `fruit.rb` files in the project: `models/healths/diets/fruit.rb` would mean "Fruit-part of the diet", while `stores/shopping/fruit.rb` would mean the fruit-shop storefront, and `models/integrations/super_sync/fruit.rb` might mean some wrappers for fruits to be synced to an external integration called "Super-sync". All three of these might refer to the top level `models/fruit.rb`

**Q: And how do we figure out, whether we should put new logic into `models/fruit.rb` or to `models/healths/diets/fruit.rb`?**

A: If you look at the mind map example, you'd see that we are narrowing the scope. So the `models/healths/diets/fruit.rb` is fruit-part of the diet-part of the health-part of a user. While the top term `models/fruit.rb` means things common for any fruit. So, you'd put nutritional diet formulas specific to fruit — to the `models/healths/diets/fruit.rb`, while the `def color` method might go into `models/fruit.rb`.

Also, if you need persistence and your model represents a table in the database, it's usually at the `model/<here>` top layer.

**Q: Do I understand correctly, that we just banned all the patterns, because we shouldn't create top level folders like `decorators`?**

A: Not really. Rails support all tools to build [decorators](https://github.com/rails/rails/issues/23824), [delegates](https://apidock.com/rails/Module/delegate), [singletons](https://ruby-doc.org/stdlib-2.5.1/libdoc/singleton/rdoc/Singleton.html) and anything you want. We didn't ban them. What we do is avoid creation of top level folders like `app/singletons`, `app/decorators`, `app/policies`, `app/delegates`, etc. and only reuse basic Rails folders like `serializers`/`controllers`. If your `models/health.rb` is a singleton, because it represents "Health concept as a whole" — perfect, include `Singleton` in it. If your `models/pages/secret_page.rb` is a decorator of your `models/page.rb` — perfect, just delegate needed methods or `delegate_missing`. But to not start to group your models by the patterns, because belonging to a pattern is just a technical detail of a specific model.

**Q: But I see `app/services`, `app/forms` and so on in your code! These are explicitly banned, right?**

A: Unfortunately, when we make a decision to remove forms or ban service objects, we can't afford to immediately refactor all our legacy 100500 classes, so we refactor them slowly and only when it's needed. So for now just treat these as legacy which we should refactor in future. Same as having the `test` folder, while we switched to `rspec` more than a year ago.

**Q: I feel like some parts are vague. I can't figure out what is 'too fat', 'too slim', 'balanced', I can't always figure out the main subject of a story with 2-3 interacting subjects and so on. What should I do?**

A: We have a pretty opinionated way on top of an opinionated framework. Many things here are subject to implicit learning and it won't be possible to give you an algorithm to follow exactly to achieve "perfect Uscreen code". This document serves as a best effort to give you a set of ideas, thoughts and examples to teach and train your neural network to learn and get a feeling of what's expected at Uscreen. In addition to that, we have specific roles, such as Development Lead, Head of Development, Director of Engineering — who review the incoming code from the expected tech vision standpoint and will correct you if you create something which is more a service rather than a business term.

**Q: You are saying that "too slim" might also a bad thing, same as "too fat". Does this mean that I should never create new models with 1-2 functions if I'm adding a new domain area?**

A: Not really. Obviously, when you add a new fresh class, most of the time it will be very slim and contain 1 function, because you just "started" a domain model and it will become larger later. "Not too slim" is more about an unhealthy degree of separation during planning the architecture: when you start to extract every action/every verb to a separate class, instead of extending the existing domain. "Too slim" doesn't mean "contains few lines of code", we don't calculate lines of code at all here. "Too slim" means we missed a good abstraction, so we end up with too much "glue" code, too many code jumps to grasp the whole picture and understand the concept.

**Q: I believe that Rails is wrong, dry-rb is great, service object is an awesome pattern, controllers must only do routing, validations belong to a schema and not to the model, persistence and model are different things, you confuse view model layer with controller, mixing a repository into an entity is bad for you and Hanami rocks! Let's discuss and let me teach you all the right things, because you are doing all wrong!**

A: We hear you, but code and software vision is an opinionated thing and we love Rails. Our choice is to follow simplest and the most described way of doing things and we generally happy with it. We believe that debate is useful in [early states](https://en.wikipedia.org/wiki/Disagree_and_commit) of decision-making while harmful after a decision has been long made. We should discuss the minor details, e.g. "virtual model attribute vs controller concern", but not the main vector itself. Treat it as a happy pass to make everyones life easier, unify our code base, and simplify new teammates onboarding.

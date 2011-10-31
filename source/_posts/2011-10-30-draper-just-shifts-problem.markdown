---
author: "Felix Schäfer"
layout: post
title: "Draper just shifts the problem"
date: 2011-10-30 18:25
comments: true
categories:
  - Ruby on Rails
  - Draper
---

__tl;dr__: [Draper][] wraps the Rails "helper dumps" in objects resulting in namespaced
dumps you have to remember calling. Yay (not).

[Draper]: https://github.com/jcasimir/draper/ "Draper on GitHub"

_Disclaimer_: I'm no expert on OOP (apart from the basic class at the university
that teaches that "putting code in classes" is good) nor Rails (I learnt ruby and
Rails by myself roughly 2 years ago while tinkering with Redmine (hint: not the
nicest code in the Rails world)), so don't take my opinion for granted, I just
hope to fuel the discussion a little.

Flashback to pre-Draper times: everyone has a feeling of sorts that the Rails
helpers are probably a bad idea, but they still become the dumping place for
any formatting methods and other stuff you don't know where else to put. Formatting
methods landed there because having them in your ActiveRecord models made the
models look bloated which made you feel even worse than putting them in the helpers
and/or because what you wanted the method do do wouldn't work in the model
because models miss the ActionView helpers (`link_to` et al.). Over time, the helpers
would get bloated, you'd have to add them to an increasing number of views,
maybe even include them in the controllers because you'd put some controller
logic in there or needed view logic in the controller (I'm looking at you,
`#to_xml`, but that's a different matter). In the end, everyone would feel bad about
the helpers becoming so big and jumbled together, but it still was better than having all this
stuff in your models or controllers (or views!) and more convenient because you
could even add all helpers to all view and be done with it (provided you didn't
have any methods with the same names in different helpers).

Enter [Draper][]. It has a nice sounding name, claims to solve an itch Rails
developers have, and uses a catchy sounding OOP pattern everyone has heard
about some time or another as selling point. Problem solved, right? I thought so too at first
(beware of the buzzwords!), but [Andreas][] ([@mediafinger][]) and Uygar's
([@uygar_gg][]) [presentation][] at the RailsCamp Hamburg made me realize that the
lurking unease I had about Draper is justified after all. Draper just wraps your
helpers in objects (something you could have easily done before but rightfully
didn't) thus solving the cross-helper method name collision problem, at the cost of
just a little less hassle than doing it yourself would have been while masking
the pain you'd have had making decorators yourself.

[Andreas]: http://mediafinger.com/ "Andreas Finger's homepage"
[@mediafinger]: https://twitter.com/mediafinger "Andreas Finger on Twitter"
[@uygar_gg]: https://twitter.com/uygar_gg "Uygar Gomez on Twitter"
[presentation]: https://github.com/mediafinger/rails_presenter_with_draper "Rails presenter with Draper"

Let's have a look at an example, first without Draper:

``` ruby
class User < ActiveRecord::Base
end

class UserController < ApplicationController
  helper :user
  include UserHelper

  before_filter :find_user, :only => [:show, :update, :edit, :destroy]

  private
  def find_user
    @user = User.find(params[:id])
  end
end

module UserHelper
 def twitter_link_for_user(user)
   link_to "https://twitter.com/#{user.twitter_nickname}"
  end
end
```

Not really pretty because you have some function `twitter_link_for_user` that
looks like it should rather belong to the `User` class (the _for_user_ part of
the method name should be enough of a hint) rather than lurch in the helper,
 and you have included the helper in the controller, for example to include the
 twitter link in some `#to_xml` call. If I'm not completely mistaken, Rails helpers
 are view helpers, including them in your controllers puts view logic in them,
 which kinda renders the whole MVC pattern moot and should be considered a code smell.

Now the same example with Draper:

``` ruby
class User < ActiveRecord::Base
end

class UserDecorator < ApplicationDecorator
  decorates :user

  def twitter_link
    h.link_to "https://twitter.com/#{user.twitter_nickname}"
  end
end

class UserController < ApplicationController
  before_filter :find_user, :only => [:show, :update, :edit, :destroy]

  private
  def find_user
    @user = UserDecorator.find(params[:id])
  end
end

module UserHelper
end
```

You can now get the twitter link for the user by calling `@user.twitter_link`
and you were able to stop including the helper in the controller, you can even
drop the `helper` call in the controller if your helper becomes empty. You feel
all fuzzy inside and are comfortable with your code again. The helper is empty
and everything is neatly organized in objects, so it's all OOP and must thus be
good, right? I don't think so.

Let me rephrase the above without using Draper (I'm aware Draper does a little
bit more than that, but bear with me):

``` ruby
class User < ActiveRecord::Base
end

class UserDecorator
  include ActionView::Helpers

  def initialize(user)
    @user = user
  end

  def twitter_link
    link_to "https://twitter.com/#{@user.twitter_nickname}"
  end
  
  def method_missing
    # try calling on @user what we didn't implement here
  end
end

class UserController < ApplicationController
  before_filter :find_user, :only => [:show, :update, :edit, :destroy]

  private
  def find_user
    @user = UserDecorator.new(User.find(params[:id]))
  end
end

module UserHelper
end
```

How does it look without the nice syntax Draper brings? There's an explicit
call to `ActionView::Helpers` in something directly called by the controller.
It doesn't feel so warm and fuzzy after all, does it? Here's the slide from the
aforementioned presentation which made me realize that you're effectively
squeezing view code between your model and controller:

<a href="/uploads/2011/10/kill_your_helpers-slide_10-960x.png" title='Slide 10 of the presentation "Rails presenter with Draper"'>{% img center /uploads/2011/10/kill_your_helpers-slide_10-300x.png 'Slide 10 of the presentation "Rails presenter with Draper"'%}</a>

Now if you applied the decorator to your object only "after" the controller,
I wouldn't mind (provided said decorator contains view logic only). Sadly, that's
not the case in the code examples provided by Draper.

The approach presented above has more pitfalls still, and one that isn't apparent
from the code above is that you're working on "fully" decorated objects
in the controller. Apart from bringing view logic into your controller, this
also is problematic with another feature of Draper which allows to white- or
blacklist functions of the decorated object. Great feature, you decide to blacklist
write operations to your object, including save, but that means for "write" actions
(create and update) you can't use the decorated object in your controller logic.
No big deal, you just use the plain object and decorate only if the operation
fails and you have to present the user with the form for the object (this would
be the right way to apply view decorators, by the way). Now for whatever reason,
for example for access control, you stick a method into your decorator (it's
only of interest to instances of that particular class but isn't business logic,
so you don't put it in the helper nor in the object but into the decorator) and
now need the decorated object before you call save on it. You now have to juggle
a decorated and an undecorated instance of the object at the same time. Bummer.

Let's recap what bothers us with the Rails helpers and what Draper can do for
us. Does it avoid cross-helper method name collisions? Sure. Is it "more OOP"?
You're explicitly calling your helper methods on an object rather than including
a misnamed hodgepodge of methods to be able to call them in views, so let's
say yes here too. Does it take this jumbled together mess out of the helpers?
It does, and sticks this same mess but with better names into an object. I don't
consider a namespaced mess better than a non-namespaced one, so this is a draw.
Does it keep view logic out of your controllers? Nope, I'd even say it makes
things worse. Does it keep view logic out of your models? Yeah, but so do helpers.
Does Draper give us more than a buzzword we can add to the list of "OOP things"
our code does? I don't think so, and you probably shouldn't either.

How can the Rails helper situation be improved? I'm not sure I have a good answer
to that, but if you think your helpers are too bloated, try splitting them
into smaller topical modules. If you think your helper methods should really
be methods on your business objects but don't want to litter your model definitions
with view logic or include the ActionView helpers in them (and you shouldn't!),
then by all means use decorators, but don't use _them_ as yet another dumping ground!
Feel free to use more than one decorator on business object instances, for example
a `TwitterUser`, a `GooglePlusUser` and a `XingUser` (yeah, decorators don't need
to be called `SomeDecorator`). Sure, this results in a call akin to
`TwitterUser.new(GooglePlusUser.new(XingUser.new(@user)))`, but at least it is
very clear what each decorator does and it makes it easier to swap one for another
or refactor any of them. Make also sure to know which decorator belongs to which
MVC step, you can use decorated objects in your controller logic, but decorate
your objects with view logic only at the end of or after your controller logic!
Sure, that's not as easy just dumping everything into the helpers or into Draper
decorators (I know you can apply multiple Draper decorators on an object and thus
avoid bloating them, the code examples and the fact that the decorated objects
seem to get used in controller logic doesn't indicate that this was the authors
primary intentions. Furthermore, `FooDecorator.find` doesn't make sense for a 
"proper"/reusable decorator), but no one ever said programming was easy :-)
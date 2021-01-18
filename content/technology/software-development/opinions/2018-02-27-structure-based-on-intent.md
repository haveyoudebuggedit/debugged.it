---
slug: structure-based-on-intent
guid: https://pasztor.at/blog/structure-based-on-intent/
authors:
- janos
categories:
- Technology
date: "2018-02-27T00:00:00Z"
publishDate: "2018-02-27T00:00:00Z"
summary: When reviewing code I often see folders like model, view, controller, form, etc. There is a way to make your folder names talk, but this ain't it.
images:
- posts/structure-based-on-intent.png
preview: posts/structure-based-on-intent.jpg
tags:
- Software Development
title: Structure your program based on Intent, not Architecture
---

Let's set aside the fact that I think MVC is a horrible design pattern (we'll talk about that some other day) and focus
solely on what happens if you have a standard MVC directory structure:

<figure>
<ul class="tree">
  <li class="tree__item tree__item--folder">
    src
    <ul class="tree__subtree">
      <li class="tree__item tree__item--folder">controller</li>
      <li class="tree__item tree__item--folder">model</li>
      <li class="tree__item tree__item--folder">view</li>
    </ul>
  </li>
</ul>
</figure>
  
On the surface it sounds good, right? You put your controllers in the controller folder, the models in the model folder,
and so on. This works reasonably well for a small sample application like a blog, since you'll have maybe 5 controllers,
6 models, and so on.

However, when you you work on a larger application, or you follow the concept of
[one controller, one action](/blog/one-controller-one-action), these number of files in these folders escalates quickly
and becomes a tangled mess of naming problems.

{{% tip %}}
**Recommended talk:** [Robert "Uncle Bob" Martin - Architecture: The Lost Years](https://www.youtube.com/watch?v=HhNIttd87xs)
{{% /tip %}}

Let's look at an example of a social media project. You have *Walls*, *WallPosts*, *Comments*, *PrivateConversations*, 
*PrivateConversationMessages*, and a bunch more stuff. Using a *"classic"* arrangement, we'll have quite a sizable
directory tree, even without the previously mentioned method:

<figure>
<ul class="tree">
  <li class="tree__item tree__item--folder">
    src
    <ul class="tree__subtree">
      <li class="tree__item tree__item--folder">
        controller
        <ul class="tree__subtree">
          <li class="tree__item tree__item--file">
            WallController
          </li>
          <li class="tree__item tree__item--file">
            WallPostController
          </li>
          <li class="tree__item tree__item--file">
            CommentController
          </li>
          <li class="tree__item tree__item--file">
            PrivateConversationController
          </li>
          <li class="tree__item tree__item--file">
            PrivateConversationMessageController
          </li>
        </ul>
      </li>
      <li class="tree__item tree__item--folder">
        model
        <ul class="tree__subtree">
          <li class="tree__item tree__item--file">
            WallModel
          </li>
          <li class="tree__item tree__item--file">
            WallPostModel
          </li>
          <li class="tree__item tree__item--file">
            CommentModel
          </li>
          <li class="tree__item tree__item--file">
            PrivateConversationModel
          </li>
          <li class="tree__item tree__item--file">
            PrivateConversationMessageModel
          </li>
        </ul>
      </li>
      <li class="tree__item tree__item--folder">
        view
        <ul class="tree__subtree">
          <li class="tree__item tree__item--file">
            ...
          </li>
        </ul>
      </li>
    </ul>
  </li>
</ul>
</figure>

Now, this is just a simplified example, in your real world application you'll have a *lot* more controllers, models
and views. If you structure based on what your application is made of (models, views, controllers) your directory
structure becomes completely unusable after a certain point.

You can, of course, use the search function of your IDE, but an overloaded directory structure results in the feeling
that *you have too many classes*. This feeling in turn results in a **fear of adding new classes**, your dev will try to
cram a new feature into an existing class even if it does not strictly belong there, resulting in a lot of [Single
Responsibility Principle violations](/blog/clean-code-responsibilities). Additionally, this structure makes it
incredibly hard for a new developer on the team to get a picture of what is what.

## Structure based on intent

If we look closely at our controllers, we can draw lines between what belongs together and what doesn't. For example, from a *business perspective* the Wall seems to be a concept that is quite clear cut: people can have walls, write posts, and comment on said posts. So let's put everything related to a wall into one folder. Similarly, with private conversations:

<figure><ul class="tree">
  <li class="tree__item tree__item--folder">
    src
    <ul class="tree__subtree">
      <li class="tree__item tree__item--folder">
        wall
        <ul class="tree__subtree">
          <li class="tree__item tree__item--file">
            CommentController
          </li>
          <li class="tree__item tree__item--file">
            CommentModel
          </li>
          <li class="tree__item tree__item--file">
            WallController
          </li>
          <li class="tree__item tree__item--file">
            WallPostController
          </li>
          <li class="tree__item tree__item--file">
            WallModel
          </li>
          <li class="tree__item tree__item--file">
            WallPostModel
          </li>
        </ul>
      </li>
      <li class="tree__item tree__item--folder">
        conversation
        <ul class="tree__subtree">
          <li class="tree__item tree__item--file">
            PrivateConversationController
          </li>
          <li class="tree__item tree__item--file">
            PrivateConversationModel
          </li>
          <li class="tree__item tree__item--file">
            PrivateConversationMessageController
          </li>
          <li class="tree__item tree__item--file">
            PrivateConversationMessageModel
          </li>
        </ul>
      </li>
      <li class="tree__item tree__item--folder">
        ...
      </li>
    </ul>
  </li>
</ul></figure>

Still not great, but *better*. Now the directory structure can be extended without the fear of too many classes, and if
we are looking for something it is quite clear where we can find it.

Before we continue, let's clarify one thing: these "modules" are not *independent*. Sometimes they may have 
cross-dependencies that may need to be addressed if you want to ship the modules separately, but that's the topic of a
different article.

You may notice that now everything within a module is thrown in a single directory. This is good because it will
(hopefully) prevent you from adding too many things into one module. However, if you are like me, you still prefer
having some structural building blocks around, so let's bring back the previous directory structure but one level lower:

<figure>
<ul class="tree">
  <li class="tree__item tree__item--folder">
    src
    <ul class="tree__subtree">
      <li class="tree__item tree__item--folder">
        wall
        <ul class="tree__subtree">
          <li class="tree__item tree__item--folder">
            controller
            <ul class="tree__subtree">
              <li class="tree__item tree__item--file">
                CommentController
              </li>
              <li class="tree__item tree__item--file">
                WallController
              </li>
              <li class="tree__item tree__item--file">
                WallPostController
              </li>
            </ul>
          </li>
          <li class="tree__item tree__item--folder">
            model
            <ul class="tree__subtree">
              <li class="tree__item tree__item--file">
                CommentModel
              </li>
              <li class="tree__item tree__item--file">
                WallModel
              </li>
              <li class="tree__item tree__item--file">
                WallPostModel
              </li>
            </ul>
          </li>
          <li class="tree__item tree__item--folder">
            view
            <ul class="tree__subtree">
              <li class="tree__item tree__item--file">
                ...
              </li>
            </ul>
          </li>
        </ul>
      </li>
      <li class="tree__item tree__item--folder">
        conversation
        <li class="tree__item tree__item--folder">
          ...
        </li>
      </li>
    </ul>
  </li>
</ul>
</figure>

Small, easy to navigate and easy to read chunks of code. You can, of course, continue adding business structures in
a hierarchic fashion as deep as you feel comfortable with. I would advise that you keep it below 3-5 levels to make
navigation easy.

{{% tip %}}
**Tip:** MVC is not suitable to be your overarching design pattern. Instead, I would recommend taking a look at [Entity-Boundary-Interactor](http://ebi.readthedocs.io/en/latest/)
{{% /tip %}}

To sum it up, **your outer most folder structure should be based on business concepts (intents)**, not the design pattern
you chose to use.

{{% tip %}}
**Recommended reading:** [One Controller, One Action](/blog/one-controller-one-action)
{{% /tip %}}

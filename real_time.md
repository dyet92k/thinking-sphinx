---
layout: en
title: Real-time Indices
---

## Real-time Indices

Historically, Thinking Sphinx has supported SQL-backed indices. However, in recent releases (since 3.0.4), real-time indices are supported as well. Real-time indices can be updated on a per-record basis, which removes the need for delta indices. The trade-off is that they're usually a bit slower to populate (as each record must be iterated through, instead of bulk inserts via SQL queries).

The documentation here is now pretty good at distinguishing any different behaviours between the two indexing approaches, and there is [a blog post](http://freelancing-gods.com/2013/07/22/rewriting-thinking-sphinx-introducing-realtime-indices.html) from when the feature was first released which covers the basics quite well.

However, if you have used SQL-backed indices before and want to change, this page is the best place to understand what's different.

### Rake Tasks

With real-time indices, you should never use `ts:index` or `ts:rebuild`. Their functional equivalents are `ts:generate` and `ts:regenerate` (as covered on [the rake tasks page](rake_tasks.html)). You only need to run `ts:regenerate` when you've added an index, removed an index, or edited the structure of an index (e.g. added/removed/modified fields or attributes).

### Indexes, Fields & Attributes

Real-time index definitions operate in a slightly different context: your model, rather than your model's database table. The implications of this are as follows:

#### Methods and values

Field and attributes refer to methods rather than columns and associations. You can chain method calls, but each intermediate method must refer to a single object, and the final item in the chain must return the data in the right format. Thus, fields must return strings, attributes must return objects of their specified type, or arrays of such items, if they're a multi-value array.

#### Attribute types

Because the database isn't available as a reference, and Ruby is dynamically typed, attributes must have their type explicitly set, and if they're a multi-value attribute, that is required as well.

{% highlight ruby %}
# in a SQL-backed index
has tags.id, :as => :tag_ids

# can, in a real-time index, use the automatic association method:
has tag_ids, :type => :integer, :multi => true
{% endhighlight %}

#### Custom Methods

If you were using SQL snippets to modify column data, or are aggregating values in some way, the real-time approach is to instead define a method in your model that does the same:

{% highlight ruby %}
# in a SQL-backed index:
indexes comments.text, :as => :comments

# the real-time approach…
# in your model:
def comment_texts
  comments.collect(&:text).join(' ')
end

# and in your index:
indexes comment_texts
{% endhighlight %}

### Model Callbacks

To ensure changes to your model instances are reflected in Sphinx, you'll need to add a callback:

{% highlight ruby %}
# if your model is app/models/article.rb:
after_save ThinkingSphinx::RealTime.callback_for(:article)
{% endhighlight %}

If you want changes to associated data to fire Sphinx updates for a related model, you can specify a method chain for the callback.

{% highlight ruby %}
# in app/models/comment.rb, presuming a comment belongs_to :article
after_save ThinkingSphinx::RealTime.callback_for(
  :article, [:article]
)
{% endhighlight %}

The first argument, in all situations, should match the index definition's first argument: a symbolised version of the model name. The second argument is a chain, and should be in the form of an array of symbols, each symbol representing methods called to get to the indexed object (so, an instance of the Article model in the example above).

### Testing

Because real-time indices are updated through your Ruby app, rather than by interacting with the database, you can use transactional fixtures in your tests. However, you'll probably want to disable Sphinx and the callbacks in unit tests. Here's an example for how I set things up with RSpec (where callbacks and Sphinx are only enabled for request specs):

{% highlight ruby %}
RSpec.configure do |config|
  config.use_transactional_fixtures = true

  config.before :each do |example|
    if example.metadata[:type] == :request
      ThinkingSphinx::Test.init
      ThinkingSphinx::Test.start index: false
    end

    configuration = ThinkingSphinx::Configuration.instance
    configuration.settings['real_time_callbacks'] =
      (example.metadata[:type] == :request)
  end

  config.after(:each) do |example|
    if example.metadata[:type] == :request
      ThinkingSphinx::Test.stop
      ThinkingSphinx::Test.clear
    end
  end
end
{% endhighlight %}
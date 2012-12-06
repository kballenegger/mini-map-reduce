# MiniMapReduce

MiniMapReduce is a library that provides a simple DSL for data-processing operations in Ruby. This is not meant to be a fast engine, and is not meant to compete with Hadoop & friends. Rather, it's meant to be an expressive way to process data in your favorite programming language. Use it to explore and prototype.

## Installation

Add this line to your application's Gemfile:

    gem 'mini-map-reduce'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install mini-map-reduce

## Usage

MiniMapReduce is invoked by creating a pipeline, and within that pipeline, using seeds, maps, and reductions.

An example is worth a thousand words, so here goes:

```ruby
require 'mini-map-reduce'

comments = [
  {author: 'kenneth', score: 20},
  {author: 'joe', score: 2},
  {author: 'kenneth', score: 19},
  {author: 'bob', score: 33}
]

MiniMapReduce.process do

  seed { comments.pop }

  map {|comment| emit(comment[:author], :score => comment[:score], :count => 1) }

  reduce do |author, comments|
    result = {score: 0, count: 0}
    comments.each do |e|
      result[:score] += e[:score]
      result[:count] += e[:count]
    end
    result
  end

  translate do |id, result|
    result[:average] = result[:score] / result[:count]
    result
  end

  dump {|id,r| puts "#{id} averages #{r[:average]}" }

end
```

A more complex example:

```ruby
require 'mini-map-reduce'

# let's say cars.json contains records which look like this:
#   {"name":"Audi R8","country":"DE","score":20,"votes":3}

data = File.open('cars.json', 'r')
out = File.open('out.json', 'w')

# let's say we want to find out which country makes the best cars
MiniMapReduce.process do

  # read every line from the data file, and use it as seed data
  seed { l = data.gets ? JSON.parse(l) : nil }

  # map data to emit scores by country (essentially a group by)
  map do |car|
    emit(car['country'],
         :score => car['score'],
         :votes => car['votes'],
         :average_sum => car['score'] / car['votes'],
         :count => 1)
  end

  # reduce per country
  # cars is an Array of the objects emitted above
  # this block is executed for each value of country
  reduce do |country, cars|
    result = {
      :score => 0,
      :votes => 0,
      :average_sum => 0,
      :count => 0
    }

    averages_equalized
    cars.each do |car|
      result[:score] += car[:score]
      result[:votes] += car[:votes]
      result[:count] += car[:count]
      result[:average_sum] += car[:average_sum]
    end

    result
  end

  # calculate the final averages and cleanup the output
  translate do |id, result|
    result[:average_raw] = result[:score] / result[:votes]
    result[:average_equalized] = result.delete(:average_sum) / result[:count]
    result[:id] = id
    result
  end

  # dump the results to disk
  dump {|id,r| out.puts(r.to_json) }

end

data.close; out.close
```

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

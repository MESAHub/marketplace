---
title: "Times in the history files"
date: 2016-09-03T10:49:28-04:00
author: Bill Wolf
---

Just a reminder -- the mesa/star history files (e.g., `LOGS/star.log`) contain a line for each step,
including steps that were subsequently "thrown away" for some reason such as restart, retry,
or backup. So when you make a plot in which you don't want to include discarded steps, you
need to remove them yourself. I do this by using model numbers -- i.e., for each line, I discard
any previous lines in the log that have a model number >= this one.

My history scripts take care of this for you -- but when you write your own, it is up to you.

Here's my little ruby routine (from `mesa/star/test/star_history/lib/history_logs.rb`)
All of the columns of data are in the vector @columns, and this routine prunes them all
to remove all kinds of backups based on model_number.

Cheers,
Bill

```ruby
def remove_backups(dbg)
    # make a list of the ones to be removed
    lst = []
    n = @model_number.size
    (n-1).times do |k|
        lst << k if @model_number[k] >= @model_number[k+1..-1].min
    end
    return if lst.length == 0
    puts "remove #{lst.length} models because of backups" if dbg
    lst = lst.sort
    @columns.each { |vec| vec.prune!(lst) }
    num_models = @model_number.length
end
```
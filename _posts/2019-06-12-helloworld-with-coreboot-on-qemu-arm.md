---
layout: post
title:  Code Reading of Coreboot Project for  QEMU/ARM
date:   2019-05-12 9:00 +0900
categories: coreboot
---

GSoC2019 の一環として参加している [Coreboot][coreboot] プロジェクトのコードリーディングをしました。Coreboot では QEMU 上でのシミュレーションをサポートしており、その中の QEMU/ARM 上で実行する際の大まかな流れを掴みました。マシンは vexpress-a9、CPU は cortex-a9 をサポートしています。つまり coreboot のルートディレクトリにおいて `qemu-system-arm -bios ./build/coreboot.rom -M vexpress-a9 -nographic` と実行した際に、Linux Kernel や GRUB2 などの任意のペイロードに処理を渡すまでを解説します。

## Architecture





You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[coreboot]: https://www.coreboot.org/

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

ゆくITのながれは絶えずして、しかももとの技術にあらず。
淀みに浮かぶシステムは、かつ消えかつ結びて、久しくとゞまりたるためしなし。

ことしげなるまゝに、日くらしキーボードに向かひて、
モニタにうつりゆくよしありごとをそこはかとなく書き付くれば、
あやしうこそ物狂ほしけれ。

<div class="posts_list">
  <ul>
    {% for post in site.posts %}
      <li>
        {{ post.date | date: "%Y-%m-%d" }}
        <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
      </li>
    {% endfor %}
  </ul>
</div>

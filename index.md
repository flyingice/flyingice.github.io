#### 文章列表:
<div class="post-list-body">
    <div post-cate="All">
        <ul>
            {% for post in site.posts %}
            <li>{{ post.date | date:"%Y-%m-%d" }} <a href="{{ post.url }}"> {{ post.title }}</a></li>
            {% endfor %}
        </ul>
    </div>
</div>

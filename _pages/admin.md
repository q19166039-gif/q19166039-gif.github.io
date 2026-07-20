---
layout: single
title: 文章管理
permalink: /admin/
author_profile: false
sitemap: false
search: false
---

此页面用于管理博客文章。编辑和删除操作会跳转到 GitHub，并由 GitHub 登录及仓库写权限保护；本站不会保存访问令牌。

<div class="notice--info">
  删除文章会在仓库中创建一次删除提交。操作完成并部署后，文章才会从网站消失。
</div>

<table>
  <thead>
    <tr>
      <th>文章</th>
      <th>发布日期</th>
      <th>操作</th>
    </tr>
  </thead>
  <tbody>
  {% for post in site.posts %}
    <tr>
      <td><a href="{{ post.url | relative_url }}">{{ post.title }}</a></td>
      <td>{{ post.date | date: "%Y-%m-%d" }}</td>
      <td>
        <a href="https://github.com/{{ site.repository }}/edit/{{ site.repository_branch | default: 'main' }}/{{ post.path }}" target="_blank" rel="nofollow noopener noreferrer">编辑</a>
        ·
        <a href="https://github.com/{{ site.repository }}/delete/{{ site.repository_branch | default: 'main' }}/{{ post.path }}" target="_blank" rel="nofollow noopener noreferrer">删除</a>
      </td>
    </tr>
  {% endfor %}
  </tbody>
</table>

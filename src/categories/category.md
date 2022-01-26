---
layout: default
title: Posts in category :prototype-term
prototype:
  term: category
---

<div class="py-8">
  <div class="relative px-4 sm:px-6 lg:px-8">
    <div class="text-lg max-w-prose mx-auto">
      <h1>
        <span
              class="block text-base text-center text-sky-600 font-semibold tracking-wide uppercase">posts under the category
        </span>
        <span
              class="mt-2 block text-3xl text-center leading-8 font-extrabold tracking-tight text-gray-900 sm:text-4xl">
          {{ page.category }}
        </span>
      </h1>
      <div class="mt-2 flex gap-2 justify-center">
        {% for category in page.categories %}
        <a href="/categories/{{category}}">
          <span
                class=" inline-flex items-center px-3 py-0.5 rounded-full text-sm font-medium bg-red-100 text-red-800">
            {{ category }}
          </span>
        </a>
        {% endfor %}
      </div>
      <p class="mt-8 text-xl text-gray-500 leading-8">{{ page.intro }}</p>
    </div>
    <div class="mt-6 prose prose-sky prose-lg text-gray-500 mx-auto">


      <ul>
        {% for post in paginator.documents %}
        <li>
          <a class="hover:underline" href="{{ post.url }}">{{ post.date | date: "%Y-%m-%d" }} - {{ post.title }}</a>
        </li>
        {% endfor %}
      </ul>

    </div>
  </div>
</div>

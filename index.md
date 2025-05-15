---
layout: default
title: <Angela Voo>
---

## Fast Robots - MAE 4190


![Profile Picture]({{ "assets/images/outside_headshot.JPG" | relative_url }}){: class="profile-image"}
 
My name is Angela Voo, and I am a senior Mechanical Engineering senior at Cornell University with a focus in robotics. Some hobbies I have are playing frisbee, dabbling in arts and crafts, nature photography and playing piano. This website will showcase the projects done in Fast Robots in Spring 2025.


<!--Take a look at <a href="{{ "/projects/" | relative_url }}">my projects</a> and <a href="{{ "/cv/" | relative_url }}">CV</a>.-->

Labs (click on text):

<!--<p><a href="{{ "/lab1b/" | relative_url }}">Lab 1B</a></p>-->

<div class="container">
    <div class="project-gallery">
        {% for project in site.projects %}
          <div class="gallery-item">
            <a href="{{ project.url | relative_url }}">
              <p>{{ project.title}}</p>
            </a>
          </div>
        {% endfor %}
    </div>
  </div>




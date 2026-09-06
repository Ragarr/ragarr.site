---
title: Static Website with HTML, CSS, and JavaScript
description: Building a static personal website to showcase my CV and portfolio before migrating to Jekyll.
date: 2024-06-23
tags: [HTML, CSS, JavaScript, Web Development]
categories: [Coursework, Web]
pin: false
toc: true
lang: en
image:
  path: /media/2024-06-23-Static Website/portada.png
  alt: Static website cover
---

## About the project

This was a **static website** I created to showcase my CV and portfolio before migrating to Jekyll (my current site). It was built using plain **HTML, CSS, and JavaScript** and hosted on GitHub Pages.

You can still view the original website here: [Portfolio link](https://ragarr.github.io/Portfolio/)

<iframe
  src="https://ragarr.github.io/Portfolio/"
  style="width:70%; height:300px; border:none; overflow:hidden; border-radius: 10px"
></iframe>

## Website design
The design was simple and minimalistic: a **single-page layout** focused on clarity and readability.

### Information layout

The site was split into **two columns**:  
- Left column → personal info and contact details.  
- Right column → CV and portfolio.  

The right side supported vertical scrolling to navigate between sections, while the left remained fixed for easy reference.

### Responsive design

A key goal was to ensure **responsiveness** across devices. This was achieved with CSS **media queries** and a **grid layout**, which collapsed into a single-column layout on smaller screens.

```css
.columns{
    max-width: 1080px;
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    column-gap: 5rem;
    z-index: 2;
}
@media (max-width: 768px) {
    .columns {
        display: flex;
        flex-direction: column;
    }
    ...
}
```

### Animations, effects, and aesthetics

To make the page more dynamic, I added **CSS and JavaScript animations**:
- A **light-follow effect** where a radial gradient followed the mouse cursor.

```javascript
document.addEventListener("mousemove", function (e) {
    const innerRadius = "0";
    const outerRadius = "300px";
    const xPercent = (e.clientX / window.innerWidth) * 100;
    const yPercent = (e.clientY / window.innerHeight) * 100;
    const light = document.querySelector(".light");
    let new_bg = `radial-gradient(circle at ${xPercent}% ${yPercent}%, ${circleColor} ${innerRadius}, transparent ${outerRadius} )`;
    light.style.background = new_bg;
});
```

![Light animation](/media/2024-06-23-Static Website/fig1.gif)

- **Hover animations**: highlighting hovered elements while dimming others.

```javascript
blocks.forEach((project) => {
    project.addEventListener("mouseover", function (e) {
        if (window.innerWidth < 768) return;
        let target = e.target;
        while (!target.classList.contains("block")) {
            target = target.parentElement;
        }
        blocks.forEach((project) => {
            if (project !== target) project.classList.add("not-hovered");
        });
        target.classList.add("hovered");
    });
});
```

![Hover animation](/media/2024-06-23-Static Website/fig2.gif)

## Conclusion

This project was a great way to **practice HTML, CSS, and JavaScript**, while learning to design a responsive, minimalistic, and interactive website.  

It also gave me a **personal online space** to share my CV and portfolio, which proved useful for showcasing my work to potential employers.  

Overall, I was very satisfied with the outcome and highly recommend similar projects to anyone looking to improve their **front-end development skills**.


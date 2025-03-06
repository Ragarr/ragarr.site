---
title: Página web estática con HTML, CSS y JavaScript
description: Creación de una página web estática con HTML, CSS y JavaScript que mostraba mi CV y portfolio antes de que me cambiara a Jekyll.
date: 2024-06-23
tags: [HTML, CSS, JavaScript, Desarrollo Web]
categories: [Proyectos, Propios]
pin: false
toc: true
image: 
    path: /media/2024-06-23-Pagina Web Estatica/portada.png
    alt: Portada de la página web estática
---

## Sobre el proyecto

Este proyecto fue una página web estática que creé para mostrar mi CV y portfolio antes de que me cambiara a Jekyll (pagina actual) . La página web estática fue creada con **HTML**, **CSS** y **JavaScript** plano y alojada en GitHub Pages.

Llevaba tiempo queriendo hacer una página web estática para mostrar mi CV y portfolio, y finalmente me decidí a hacerla. La pagina se desarrollo sin utilizar ningún framework, solo HTML, CSS y JavaScript plano, pues quería practicar y mejorar mis habilidades en estos lenguajes.

La pagina original se puede ver [aquí](https://ragarr.github.io/Portfolio/)

<iframe
  src="https://ragarr.github.io/Portfolio/"
  style="width:70%; height:300px; border:none; overflow:hidden; border-radius: 10px"
></iframe>

## Diseño de la página web
El diseño de la página web fue bastante sencillo, con un diseño minimalista y limpio, de una sola pagina.

### Presentación de la información

La pagina se divide en 2 columnas, la columna de la izquierda contiene la información ma importante, como un resumen sobre mi y mi información de contacto, mientras que la columna de la derecha contiene mi CV y portfolio.

El lateral derecho presenta un desplazamiento vertical que permite al usuario navegar por las distintas secciones de la pagina. Mientras que el lateral izquierdo se mantiene fijo en la pantalla.

### Diseño responsive

Uno de los objetivos del diseño era que la pagina fuera responsive y presentara un diseño adecuado y legible en cualquier dispositivo.

Para ello se emplearon media queries en el CSS para adaptar el diseño de la pagina a distintos tamaños de pantalla.

Asi como se decidió dividir el contenido de la pagina en 2 columnas, usando grid layout,  la cual se convierte en una sola columna en dispositivos móviles.

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

### Animaciones, efectos y estética

Para darle un toque más dinámico a la pagina se emplearon animaciones y efectos en CSS y JavaScript. 

Al mover el cursor un foco de luz sigue al cursor.

```javascript
document.addEventListener("mousemove", function (e) {
    const innerRadius = "0"; // Radio del color de acento
    const outerRadius = "300px"; // Radio total del gradiente
    const x = e.clientX; // Obtiene la coordenada X del cursor
    const y = e.clientY; // Obtiene la coordenada Y del cursor
    const width = window.innerWidth; // Ancho total de la ventana
    const height = window.innerHeight; // Alto total de la ventana
    const xPercent = (x / width) * 100;
    const yPercent = (y / height) * 100;
    light = document.querySelector(".light");
    let new_bg = `radial-gradient(circle at ${xPercent}% ${yPercent}%, ${circleColor} ${innerRadius}, transparent ${outerRadius} )`;
    light.style.background = new_bg;
});
```
![Animación de luz](/media/2024-06-23-Pagina%20Web%20Estatica/fig1.gif)

También la mayoría de los elementos de la pagina tienen una animación al hacer hover sobre ellos, esto se implemento con js para que la animación ademas de acentuar el elemento, también disminuyera la opacidad de los demás elementos.

```javascript
blocks.forEach((project) => {
    project.addEventListener("mouseover", function (e) {
        // if is in mobile, don't add the hover effect
        if (window.innerWidth < 768) {
            return;
        }

        // get the parent until its a project div
        let target = e.target;
        while (!target.classList.contains("block")) {
            target = target.parentElement;
        }
        blocks.forEach((project) => {
            if (project !== target) {
                project.classList.add("not-hovered");
            }
        });
        target.classList.add("hovered");
    });
});
```
![Animación de hover](/media/2024-06-23-Pagina%20Web%20Estatica/fig2.gif)

## Conclusión

Este proyecto me permitió practicar y mejorar mis habilidades en HTML, CSS y JavaScript, así como aprender a hacer una pagina web responsive y con un diseño minimalista y limpio.

Además, me permitió tener un espacio en la web donde mostrar mi CV y portfolio, lo cual es muy útil para mostrar mi trabajo a posibles empleadores.

En general, estoy muy satisfecho con el resultado de la pagina web y con lo que aprendí durante el proceso de creación y recomendaría a cualquiera que quiera mejorar sus habilidades en desarrollo web que haga un proyecto similar.

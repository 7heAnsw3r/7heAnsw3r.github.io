

# 7heAnsw3r - Klisé

**Klisé** es un tema minimalista de Jekyll, creado por [piharpi](https://github.com/piharpi), que permite construir un sitio personal y un blog de manera sencilla y elegante. Este proyecto está inspirado en la necesidad de compartir conocimientos y experiencias en el ámbito de la ciberseguridad y el hacking ético.

## 🌐 Demostración
Puedes ver una demostración del tema en [klise.now.sh](https://klise.now.sh).

## ✨ Características
- **Modos Claro y Oscuro** 🌒
- **Totalmente Personalizable** (usando .scss)
- **Diseño Responsive** (optimizado para escritorio, tabletas y móviles)
- **Mobile First Design**
- **Optimizado para SEO**
- **Organización de Imágenes de Publicaciones** (jekyll-postfiles)
- **Generación de Sitemap** (jekyll-sitemap)
- **Feed RSS** (jekyll-feed)
- **Resaltado de Sintaxis** (rouge)
- **Navegación Sencilla entre Publicaciones** (anterior/siguiente)
- **Sistema de Comentarios** (habilitable en frontmatter)
- **Integración con Google Analytics**
- **Minificación de HTML** (jekyll-compress-html)
- **Validado por W3C**
- **Aprobado por Lighthouse y PageSpeed**

## 🚧 Pendientes
- Integrar **PhotoSwipe** para la visualización de imágenes.
- Añadir **meta información schema.org** para un mejor SEO.
- Transformar el selector de clases a **metodología BEM**.

## 🛠 Instalación
Para ejecutar el tema en un servidor local, sigue estos pasos:

```bash
git clone https://github.com/piharpi/jekyll-klise.git
cd jekyll-klise
bundle config set force_ruby_platform true
bundle install
bundle exec jekyll serve
```


Accede a `localhost:4000` en tu navegador. ¡Ahora puedes empezar a personalizarlo!

## ⚠ Limitaciones
Ten en cuenta que el plugin **jekyll-postfiles** no es compatible con GitHub Pages. Esto puede causar problemas como rutas rotas o imágenes de publicaciones que no se cargan. Como alternativa, puedes usar servicios como **netlify.com**, **vercel.com**, **azure.com** o **surge.sh**, que son compatibles con terceros.

## 🤝 Contribuciones
Si encuentras errores tipográficos o de formato, o deseas ayudar a mejorar el tema, ¡no dudes en abrir un pull request! Por favor, revisa la guía de contribuciones antes de hacerlo.

## 📜 Licencia
Este proyecto es de código abierto y se encuentra bajo la Licencia MIT. 
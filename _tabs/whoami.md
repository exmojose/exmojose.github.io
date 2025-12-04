---
title: "Whoami"
icon: fas fa-user-shield
order: 1
hero: false
---

<div class="hero-container">

  <!-- Imagen de perfil -->
  <img src="/assets/img/avatar.jpg" alt="Foto de perfil de Exmojose" class="profile-pic">

  <!-- Texto de presentación -->
  <div class="hero-text">

    <h1>¡Hola!</h1>

    <p class="intro-text">
      Desde mi bunker digital me presento: Soy José <strong>(exmojose)</strong> y bienvenido a este espacio de curiosidad y aprendizaje. Pasa, ponte cómodo y siéntete libre de echar un vistazo a todo lo que tengo por aquí. 
      Aquí comparto mis experiencias resolviendo laboratorios CTF (Writeups), reviews de certificaciones, artículos de ciberseguridad y todo aquello que voy descubriendo mientras exploro el lado ofensivo de la ciber.
    </p>

    <p class="intro-text">
      Me apasiona la ciberseguridad, el hacking ético y la divulgación.  
      Fuera de mis obligaciones laborales, disfruto investigando nuevas técnicas ofensivas, participando en CTFs y colaborando con comunidades para seguir creciendo en el sector.
    </p>

    <p class="intro-text">
      Me encuentro en un proceso constante de aprendizaje, ampliando conocimientos y habilidades mediante cursos, certificaciones y errores, que luego se convierten en valiosas experiencias.  
      Mi camino como autodidacta me ha enseñado que los errores son simplemente lecciones disfrazadas de retos, y que pueden superarse con dos componentes: curiosidad y trabajo. 
      Acualmente cuento con las siguientes certificaciones:
    </p>

    <div class="certifications">
      <h2> Certificaciones</h2>

      <div class="cert-images">
        <figure>
          <img src="/assets/img/ejpt.jpg" alt="Certificación eJPT" title="eJPT">
          <figcaption>eJPT</figcaption>
        </figure>

        <figure>
          <img src="/assets/img/ewpt.png" alt="Certificación eWPT" title="eWPT">
          <figcaption>eWPT</figcaption>
        </figure>

        <figure>
          <img src="/assets/img/ecppt.png" alt="Certificación eCPPT" title="eCPPT">
          <figcaption>eCPPT</figcaption>
        </figure>
      </div>
    </div>

    <blockquote class="quote">
      “No hay mejor maestro que la práctica, ni mejor aprendizaje que compartirlo con otros.”
    </blockquote>

    <p class="intro-text">
      Espero que encuentres aquí algo útil, interesante o que despierte tu curiosidad. Si es así, habrá valido la pena.
    </p>

    <p class="contact">
        Puedes conctactar conmigo en Linkedin 
      <a href="https://www.linkedin.com/in/exmojose" rel="noopener noreferrer" target="_blank" class="social-icon">
        <i class="fab fa-linkedin"></i>
      </a>
    </p>

  </div>

  <p class="thanks">¡Muchas gracias por tu visita!</p>

</div>

<style>
.hero-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  margin-bottom: 2rem;
  padding: 1rem;
}

.profile-pic {
  width: 180px;
  height: 180px;
  object-fit: cover;
  border-radius: 50%;
  box-shadow: 0 6px 18px rgba(0,0,0,0.25);
  margin-bottom: 1.5rem;
  transition: transform 0.3s ease;
}
.profile-pic:hover {
  transform: scale(1.05);
}

.hero-text {
  max-width: 700px;
}

.intro-text {
  font-size: 1.1rem;
  margin-bottom: 1.3rem;
  line-height: 1.6;
}

.certifications h2 {
  margin-top: 2rem;
  margin-bottom: 0.8rem;
}

.cert-images {
  display: flex;
  justify-content: center;
  gap: 2rem;
  flex-wrap: wrap;
  margin-bottom: 1.5rem;
}

.cert-images figure {
  text-align: center;
}

.cert-images img {
  width: 120px;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  transition: transform 0.3s ease;
}
.cert-images img:hover {
  transform: scale(1.05);
}

.cert-images figcaption {
  margin-top: 0.5rem;
  font-size: 0.9rem;
  opacity: 0.8;
}

.quote {
  margin: 1.5rem 0;
  font-style: italic;
  opacity: 0.85;
}

.social-icon i {
  font-size: 1.8rem;
  color: #0077b5;
  transition: color 0.2s ease, transform 0.2s ease;
}

.social-icon i:hover {
  color: #005582;
  transform: scale(1.15);
}

.thanks {
  margin-top: 2rem;
  font-weight: bold;
}
</style>


---
title: "Whoami"
icon: fas fa-user-shield
order: 1
hero: false
---

<div class="hero-container">

  <!-- Imagen de perfil -->
  <img src="/assets/img/avatar.jpg" alt="Exmojose" class="profile-pic">

  <!-- Texto de presentación -->
  <div class="hero-text">
    <h1>👋 ¡Hola! Soy Jose</h1>
    <p class="intro-text">
      Apasionado del mundo de la <strong>tecnología</strong> y la <strong>ciberseguridad</strong>, en constante aprendizaje y compartiendo conocimientos con la comunidad.
    </p>

    <div class="certifications">
      <h2>🎓 Certificaciones</h2>
      <div class="cert-images">
        <img src="/assets/img/avatar.jpg" alt="eJPT" title="eJPT">
        <img src="/assets/img/avatar.jpg" alt="eWPT" title="eWPT">
        <img src="/assets/img/avatar.jpg" alt="eCPPT" title="eCPPT">
      </div>
    </div>

    <p class="description">
      Aquí encontrarás <strong>artículos de ciberseguridad</strong>, <strong>resoluciones paso a paso de CTFs</strong>, <strong>reviews de certificaciones</strong> y todo lo que he aprendido durante este apasionante viaje.  
      Mi objetivo es combinar <strong>experiencia técnica</strong> con <strong>pasión por enseñar</strong> para aportar valor real a la comunidad.
    </p>

    <p class="contact">
      🌐 Podéis contactarme en Redes Sociales:  
      <a href="https://www.linkedin.com/in/exmojose">LinkedIn</a> | 
      <a href="https://github.com/exmojose">GitHub</a>
    </p>
  </div>

</div>

<style>
.hero-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  margin-bottom: 2rem;
}

.profile-pic {
  width: 180px;
  height: 180px;
  object-fit: cover;
  border-radius: 50%;
  box-shadow: 0 6px 18px rgba(0,0,0,0.25);
  margin-bottom: 1.5rem;
  transition: transform 0.3s;
}
.profile-pic:hover {
  transform: scale(1.05);
}

.hero-text {
  max-width: 650px;
}

.intro-text {
  font-size: 1.2rem;
  margin-bottom: 1.5rem;
  line-height: 1.6;
}

.certifications h2 {
  margin-bottom: 0.8rem;
}

.cert-images {
  display: flex;
  justify-content: center;
  gap: 2rem;
  margin-bottom: 1.5rem;
}

.cert-images img {
  width: 120px;
  height: auto;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  transition: transform 0.3s;
}
.cert-images img:hover {
  transform: scale(1.05);
}

.description {
  margin-bottom: 1.5rem;
  line-height: 1.6;
}

.contact a {
  color: #0077b5;
  text-decoration: none;
  font-weight: bold;
  margin: 0 0.5rem;
  transition: color 0.2s;
}
.contact a:hover {
  color: #005582;
}
</style>


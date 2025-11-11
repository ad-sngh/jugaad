{{< rawhtml >}}
<style>
  .hero-section {
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    position: relative;
    overflow: hidden;
    padding: 2rem;
  }

  .hero-background {
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    opacity: 0.1;
    background-image: 
      radial-gradient(circle at 20% 50%, #fff 0.5px, transparent 0.5px),
      radial-gradient(circle at 80% 80%, #fff 0.5px, transparent 0.5px),
      radial-gradient(circle at 40% 20%, #fff 0.5px, transparent 0.5px);
    background-size: 200px 200px;
    animation: drift 20s linear infinite;
  }

  @keyframes drift {
    0% { transform: translate(0, 0); }
    100% { transform: translate(50px, 50px); }
  }

  .hero-content {
    position: relative;
    z-index: 10;
    text-align: center;
    max-width: 800px;
    color: white;
  }

  .hero-headline {
    font-size: 3.5rem;
    font-weight: 800;
    margin: 0 0 1rem 0;
    line-height: 1.2;
    letter-spacing: -1px;
  }

  .hero-subheadline {
    font-size: 1.3rem;
    margin: 0 0 2.5rem 0;
    opacity: 0.95;
    line-height: 1.6;
    font-weight: 300;
  }

  .hero-cta-group {
    display: flex;
    gap: 1rem;
    justify-content: center;
    flex-wrap: wrap;
    margin-bottom: 3rem;
  }

  .hero-cta {
    padding: 0.875rem 2rem;
    font-size: 1rem;
    font-weight: 600;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.3s ease;
    text-decoration: none;
    display: inline-block;
  }

  .hero-cta-primary {
    background: white;
    color: #667eea;
  }

  .hero-cta-primary:hover {
    transform: translateY(-2px);
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.2);
  }

  .hero-cta-secondary {
    background: transparent;
    color: white;
    border: 2px solid white;
  }

  .hero-cta-secondary:hover {
    background: rgba(255, 255, 255, 0.1);
    transform: translateY(-2px);
  }

  .scroll-indicator {
    position: absolute;
    bottom: 2rem;
    left: 50%;
    transform: translateX(-50%);
    animation: bounce 2s infinite;
    z-index: 10;
  }

  @keyframes bounce {
    0%, 100% { transform: translateX(-50%) translateY(0); }
    50% { transform: translateX(-50%) translateY(10px); }
  }

  .scroll-indicator svg {
    width: 24px;
    height: 24px;
    stroke: white;
    stroke-width: 2;
    fill: none;
  }

  @media (max-width: 768px) {
    .hero-headline {
      font-size: 2.5rem;
    }
    .hero-subheadline {
      font-size: 1.1rem;
    }
    .hero-cta-group {
      flex-direction: column;
    }
    .hero-cta {
      width: 100%;
    }
  }
</style>

<div class="hero-section">
  <div class="hero-background"></div>
  <div class="hero-content">
    <h1 class="hero-headline">Making ideas work, one jugaad at a time.</h1>
    <p class="hero-subheadline">I turn messy problems into working AI solutions. Exploring, tinkering, iterating—my way of making technology practical.</p>
    <div class="hero-cta-group">
      <a href="#work" class="hero-cta hero-cta-primary">See My Work</a>
      <a href="/jugaad/blog/" class="hero-cta hero-cta-secondary">Read the Blog</a>
    </div>
  </div>
  <div class="scroll-indicator">
    <svg viewBox="0 0 24 24">
      <path d="M12 5v14M5 12l7 7 7-7"></path>
    </svg>
  </div>
</div>
{{< /rawhtml >}}

---

## 🚀 Featured Work

Explore my journey across data, AI, and engineering:

- **[Learning Path](/jugaad/learning-path/)** – A complete roadmap from Data Analyst to ML Engineer to AI Agent Developer
- **[Blog](/jugaad/blog/)** – Deep dives into AI, machine learning, and software engineering
- **[Projects](#)** – Real-world solutions built with practical approaches

---

## 💡 What I Do

**Data & AI Engineering** – Building systems that learn, adapt, and scale

**ML Engineering** – From notebooks to production-grade models

**Software Architecture** – Clean code, smart design, practical solutions

---

## 📚 Latest Insights

Check out my [blog](/jugaad/blog/) for the latest posts on AI, machine learning, and engineering practices.

---

*Built with ❤️ using Hugo and Hextra theme*

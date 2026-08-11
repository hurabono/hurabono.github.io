# hurabono.github.io

🔗 **Live Site:** [hurabono.github.io](https://hurabono.github.io/)

Personal portfolio site built to support my job search for **Junior IT Support roles in Toronto, Canada**. It showcases my background, skills, and hands-on IT support projects — including a simulated help-desk environment ("Lumen Systems") built to demonstrate real-world ticket handling, Active Directory administration, and documentation practices.

## About This Site

This portfolio includes:

- **CV / Resume** — education, certifications, and skills relevant to IT support roles
- **About** — a short bio page
- **Lab** — tagged write-ups of my Active Directory home lab (Windows Server, domain setup, documented step-by-step)
- **Projects** — a simulated company IT support environment (ticket templates, knowledge base, org structure)
- **Blog** — notes and write-ups from my IT learning journey

## Built With

This site is built on top of the [**Astrofy**](https://github.com/manuelernestog/astrofy) template by [manuelernestog](https://github.com/manuelernestog), an open-source personal portfolio template released under the MIT License. Astrofy is powered by:

- [Astro](https://astro.build)
- [Tailwind CSS](https://tailwindcss.com/)
- [DaisyUI](https://daisyui.com/)

I customized the layout, content, and project structure to fit my own background and IT support portfolio.

## Getting Started

Clone the repo and install dependencies:

```bash
pnpm install
```

Start the local dev server:

```bash
pnpm run dev
```

The site will be available locally with hot-reload as you edit content.

## Project Structure

```
├── src/
│   ├── components/          # Site components (sidebar, header, cards, timeline)
│   ├── content/
│   │   └── blog/             # Blog posts (markdown)
│   ├── layouts/              # Base and post layouts
│   ├── pages/
│   │   ├── blog/
│   │   │   └── tag/
│   │   │       └── [tag]/
│   │   │           └── [...page].astro   # Blog posts filtered by tag
│   │   ├── lab/
│   │   │   └── tag/
│   │   │       └── [tag]/
│   │   │           └── [...page].astro   # Homelab write-ups filtered by tag
│   │   ├── about.astro       # About / bio page
│   │   ├── cv.astro
│   │   ├── index.astro
│   │   └── projects.astro
│   └── config.ts             # Site title, description, and global settings
├── public/                    # Static assets (favicon, profile image, etc.)
├── astro.config.mjs
├── tailwind.config.cjs
└── package.json
```

The **`lab/`** section documents my Active Directory home lab (Windows Server domain setup, `lumen.local`) as tagged, dated write-ups — mirroring the same tag/pagination pattern used for the blog.

## Deployment

This site is deployed via **GitHub Pages**, built from the `main` branch.

## Credits & License

This project is built on the [Astrofy](https://github.com/manuelernestog/astrofy) template, licensed under the MIT License. See [LICENSE](./LICENSE) for details.

All personal content (resume, projects, blog posts) is my own.

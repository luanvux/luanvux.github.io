# tieovi.github.io

Personal knowledge base for digital design, FPGA, ASIC, and Ethernet notes, published with [Quartz](https://quartz.jzhao.xyz/) on GitHub Pages.

Live site: <https://tieovi.github.io/>

## Write a New Article

1. Create a Markdown file inside `content/`.

   Organize notes by topic when useful:

   ```text
   content/
   |-- fpga/
   |   `-- clock-domain-crossing.md
   |-- asic/
   |   `-- synthesis-basics.md
   `-- ethernet/
       `-- ethernet-frame-format.md
   ```

   `content/index.md` is the home page; use another file for an article.

2. Start the article with frontmatter.

   ```markdown
   ---
   title: Clock Domain Crossing Basics
   description: Practical notes on synchronizers and CDC design.
   date: YYYY-MM-DD
   tags:
     - fpga
     - digital-design
   draft: true
   ---

   # Clock Domain Crossing Basics

   Write the article here.
   ```

   Use `draft: true` while writing. Change it to `draft: false`, or remove the field, when the article is ready to publish.

3. Write content in Markdown.

   Quartz supports headings, tables, code blocks, images, callouts, and wiki links such as `[[clock-domain-crossing]]`.

   Store images or supporting files under `content/` alongside the note or in a related assets folder, then reference them from the article.

4. Preview the site locally.

   From the repository root:

   ```bash
   npx quartz build --serve
   ```

   Open <http://localhost:8080/> and review the article, navigation, links, images, and code blocks.

5. Publish the article.

   Set `draft: false` or remove `draft`, then sync the site:

   ```bash
   npx quartz sync --message "Add article: Clock Domain Crossing Basics"
   ```

   This commits and pushes changes to the `v5` branch. GitHub Actions builds and deploys the site automatically.

6. Confirm deployment.

   Check the [GitHub Actions deployment workflow](https://github.com/tieovi/tieovi.github.io/actions/workflows/deploy.yml) and then visit <https://tieovi.github.io/>.

## Useful Commands

```bash
# Preview while writing
npx quartz build --serve

# Build without running a server
npx quartz build

# Commit and publish content changes
npx quartz sync --message "Add article: <article title>"
```

## Topic Tags

Prefer consistent tags across articles:

- `digital-design`
- `fpga`
- `asic`
- `ethernet`

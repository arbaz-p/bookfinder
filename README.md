# BookFinder

This document contains a small React project you can copy into a Create React App, Vite React app, or paste into CodeSandbox/StackBlitz. It implements a Book Finder using the Open Library Search API and displays results including covers and details.

---

## README — Quick Start

### Features
- Search books by title, author, or ISBN using Open Library Search API.
- Shows cover images (when available), title, author, first published year.
- Click a book to open a details panel with more information and links to Open Library pages.
- Pagination (Next / Previous) for result pages.
- Responsive grid layout and basic accessibility.
- Graceful error handling and no-results message.

### How to run locally
1. Create a new React app (Vite recommended):

```bash
npm create vite@latest bookfinder -- --template react
cd bookfinder
npm install
```

2. Replace `src/App.jsx` and `src/index.css` with the code sections below.
3. Start dev server:

```bash
npm run dev
```

## File: src/App.jsx

```jsx
import React, { useEffect, useState } from 'react';

const API_BASE = 'https://openlibrary.org/search.json';
const COVERS = (id, size = 'M') => `https://covers.openlibrary.org/b/id/${id}-${size}.jpg`;

export default function App() {
  const [query, setQuery] = useState('');
  const [field, setField] = useState('title');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [page, setPage] = useState(1);
  const [numFound, setNumFound] = useState(0);
  const [selected, setSelected] = useState(null);

  const PAGE_SIZE = 20;

  useEffect(() => {
    // Reset pagination when query changes
    setPage(1);
  }, [query, field]);

  useEffect(() => {
    if (!query) {
      setResults([]);
      setNumFound(0);
      return;
    }
    const controller = new AbortController();
    async function fetchBooks() {
      setLoading(true);
      setError(null);
      try {
        const params = new URLSearchParams();
        params.set(field, query);
        params.set('page', String(page));
        params.set('limit', String(PAGE_SIZE));
        const url = `${API_BASE}?${params.toString()}`;
        const res = await fetch(url, { signal: controller.signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        setResults(data.docs || []);
        setNumFound(data.numFound || 0);
      } catch (err) {
        if (err.name !== 'AbortError') setError(err.message);
      } finally {
        setLoading(false);
      }
    }
    fetchBooks();
    return () => controller.abort();
  }, [query, field, page]);

  function onSearch(e) {
    e.preventDefault();
    const form = e.target;
    const input = form.querySelector('input[name=q]');
    setQuery(input.value.trim());
  }

  function openDetails(book) {
    setSelected(book);
  }

  function closeDetails() {
    setSelected(null);
  }

  const totalPages = Math.ceil(numFound / PAGE_SIZE);

  return (
    <div className="app-root">
      <header className="header">
        <h1>BookFinder 📚</h1>
        <p className="subtitle">Search books via Open Library — title, author or ISBN</p>
        <form className="search" onSubmit={onSearch}>
          <select value={field} onChange={(e) => setField(e.target.value)} aria-label="Search field">
            <option value="title">Title</option>
            <option value="author">Author</option>
            <option value="isbn">ISBN</option>
          </select>
          <input name="q" defaultValue={query} placeholder="e.g. The Hobbit" aria-label="Search query" />
          <button type="submit">Search</button>
        </form>
      </header>

      <main className="container">
        {loading && <div className="info">Loading…</div>}
        {error && <div className="error">Error: {error}</div>}
        {!loading && !error && !results.length && query && (
          <div className="info">No results found for "{query}"</div>
        )}

        <div className="grid">
          {results.map((doc) => (
            <article key={doc.key} className="card" onClick={() => openDetails(doc)} tabIndex={0}>
              <div className="cover">
                {doc.cover_i ? (
                  // Use cover_i if available
                  <img src={COVERS(doc.cover_i)} alt={`Cover of ${doc.title}`} />
                ) : (
                  <div className="no-cover">No image</div>
                )}
              </div>
              <div className="meta">
                <h3>{doc.title}</h3>
                <p className="author">{(doc.author_name || []).slice(0,2).join(', ')}</p>
                <p className="year">First published: {doc.first_publish_year || 'N/A'}</p>
              </div>
            </article>
          ))}
        </div>

        {numFound > 0 && (
          <nav className="pager" aria-label="Search results pagination">
            <button onClick={() => setPage((p) => Math.max(1, p - 1))} disabled={page <= 1}>Previous</button>
            <span>Page {page} of {totalPages || 1} — {numFound} results</span>
            <button onClick={() => setPage((p) => Math.min(totalPages || p + 1, p + 1))} disabled={page >= totalPages}>Next</button>
          </nav>
        )}

        {selected && (
          <div className="details" role="dialog" aria-modal="true">
            <div className="details-inner">
              <button className="close" onClick={closeDetails} aria-label="Close details">✕</button>
              <div className="details-top">
                <div className="details-cover">
                  {selected.cover_i ? (
                    <img src={COVERS(selected.cover_i, 'L')} alt="book cover large" />
                  ) : (
                    <div className="no-cover">No image</div>
                  )}
                </div>
                <div className="details-meta">
                  <h2>{selected.title}</h2>
                  <p><strong>Author:</strong> {(selected.author_name || []).join(', ') || 'N/A'}</p>
                  <p><strong>First Published:</strong> {selected.first_publish_year || 'N/A'}</p>
                  <p><strong>Edition count:</strong> {selected.edition_count || 'N/A'}</p>
                  <p><strong>Subjects:</strong> {(selected.subject || []).slice(0,10).join(', ') || 'N/A'}</p>
                  <p><a href={`https://openlibrary.org${selected.key}`} target="_blank" rel="noreferrer">View on Open Library</a></p>
                </div>
              </div>
            </div>
          </div>
        )}
      </main>

      <footer className="footer">
        <small>Data from Open Library • Built with React</small>
      </footer>

      <style>{`
        :root{--bg:#f7f7fb;--card:#fff;--accent:#2563eb;--muted:#6b7280}
        *{box-sizing:border-box}
        body,html,#root{height:100%}
        .app-root{min-height:100vh;display:flex;flex-direction:column;font-family:Inter,ui-sans-serif,system-ui,Arial,Helvetica,sans-serif;background:var(--bg);color:#111}
        .header{padding:24px 20px 8px}
        .header h1{margin:0;font-size:28px}
        .subtitle{margin:6px 0 12px;color:var(--muted)}
        .search{display:flex;gap:8px;align-items:center}
        .search select,.search input{padding:8px;border-radius:8px;border:1px solid #e6e6ef}
        .search input{flex:1}
        .search button{padding:8px 12px;border-radius:8px;border:none;background:var(--accent);color:white;cursor:pointer}
        .container{flex:1;padding:18px}
        .info{padding:20px;background:#fff;border-radius:10px;max-width:800px;margin:10px auto;text-align:center}
        .error{color:#b91c1c;padding:12px}
        .grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(180px,1fr));gap:12px}
        .card{background:var(--card);padding:10px;border-radius:10px;box-shadow:0 1px 3px rgba(16,24,40,0.06);cursor:pointer;display:flex;flex-direction:column;min-height:220px}
        .card:focus{outline:2px solid #c7ddff}
        .cover{height:160px;display:flex;align-items:center;justify-content:center;margin-bottom:8px}
        .cover img{max-width:100%;max-height:100%;border-radius:6px}
        .no-cover{width:100%;height:100%;display:flex;align-items:center;justify-content:center;background:#f3f4f6;border-radius:6px;color:var(--muted)}
        .meta h3{font-size:16px;margin:0 0 6px}
        .meta p{margin:0;font-size:13px;color:var(--muted)}
        .pager{display:flex;gap:12px;align-items:center;justify-content:center;padding:12px}
        .pager button{padding:8px 12px;border-radius:8px;border:1px solid #e6e6ef;background:white}
        .details{position:fixed;inset:0;background:rgba(2,6,23,0.5);display:flex;align-items:center;justify-content:center;padding:20px}
        .details-inner{background:white;border-radius:10px;max-width:900px;width:100%;padding:18px;position:relative}
        .close{position:absolute;right:12px;top:12px;border:none;background:transparent;font-size:18px;cursor:pointer}
        .details-top{display:flex;gap:18px}
        .details-cover img{max-width:200px;border-radius:6px}
        .details-meta h2{margin:0 0 8px}
        .footer{padding:12px;text-align:center;color:var(--muted)}

        @media (max-width:640px){
          .details-top{flex-direction:column}
          .cover{height:200px}
        }
      `}</style>
    </div>
  );
}
```

---

## File: src/index.css

```css
/* minimal CSS reset for playgrounds */
html,body,#root{margin:0;padding:0}
```

---

## Notes & Tips
- Open Library API supports many fields — you can refine the query by adding `subject=`, `language=`, etc.
- For production, consider caching results and adding input debounce to reduce API calls.
- If you want to embed this in a single HTML file for rapid sharing, use tools like CodeSandbox or StackBlitz which can import a GitHub repo or accept files pasted in.

---



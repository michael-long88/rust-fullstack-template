# rust-fullstack-template
A template designed to provide a full stack Rust application with a(n)
- `axum` backend
- PostgreSQL database
- Vue3 TypeScript frontend

### Setup
- In this directory, clone [the backend](https://github.com/michael-long88/rust-backend-template) and [the frontend](https://github.com/michael-long88/rust-frontend-template)
- Copy the `.env.template` file to `.env` and change the variables as needed
- Run `docker-compose up` (or `docker-compose up -d`)

This will provide you with the following:
- The backend available at `:3000`
- The frontend available at `:5173`
- A PostgreSQL database available at `:5432`

When using this as a template, make sure to delete `blog.md` (or add it to your `.gitignore` file)
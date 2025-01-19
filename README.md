# Intro
This project using Vue 3.  

## Node.js & nvm Setup
We recommend [nvm](https://github.com/nvm-sh/nvm) for managing your Node.js versions.


1. **Install / Update nvm** (if you haven't yet):
   ```bash
   # macOS or Linux (bash, zsh)
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
   # then restart your shell or source ~/.bashrc, ~/.zshrc

2. **Use Node 18 (or the version specified in .nvmrc) :**
    ```sh
    nvm install 18
    nvm use 18
    ```
    If you have a .nvmrc file in the project root, simply:

    ```sh
    nvm install
    nvm use
    ```

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```

### Obsidian Sync
This is a normal Obsidian vault that you can pull from and push to GitHub. All of the vault's configurations and plugins will sync over. Additionally it can publish documents as GitHub pages to be shared freely.

#### Pulling the Vault
First clone the repository:
`git clone https://github.com/cgtopher/obsidian-sync.git`

Then open the new directory in Obsidian
![[Screenshot 2024-09-05 at 2.47.03 PM.png]]

When opened, allow the plugins to be activated


![[Screenshot 2024-09-05 at 9.39.18 AM.png]]

#### Push to GitHub

Once your ready to sync the changes, use the built in command line tool to push to GitHub

```
$> chmod +x publish.sh
$> ./publish.sh
```

#### Publish To GitHub Pages
After creating the document you want to publish, use the included plugin to create the web version of the document.

Click the 3 dots at the top right of the document and click "Export to HTML"

![[Screenshot 2024-09-05 at 2.52.36 PM.png]]

Then select a directory under `share/`

![[Screenshot 2024-09-05 at 2.51.34 PM.png]]

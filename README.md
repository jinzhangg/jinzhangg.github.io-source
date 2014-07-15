How to update blog:
1. Create a new file under /content/posts using RST format.
2. Run "pelican content" to regenerate the static HTML pages.
3. Optional run make devserver to have Pelican run a dev server located at localhost:8000
4. The generated HTML files is located in the /output directory.
5. While inside the /output directory, commit the changes to git then run "git push origin master", which will push the changes to jinzhangg.github.io repository.

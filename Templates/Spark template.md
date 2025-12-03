<%*
const prefix = "AS-";
const pad = n => n.toString().padStart(2, "0");

const files = app.vault.getMarkdownFiles()
  .map(f => f.basename)
  .filter(name => name.startsWith(prefix));

let maxNum = 0;
for (const name of files) {
  const m = name.match(/^AS-(\d+)/);
  if (m) {
    const num = parseInt(m[1], 10);
    if (num > maxNum) maxNum = num;
  }
}

const next = pad(maxNum + 1);

tR += `${prefix}${next} `;
%>
<% tp.file.cursor() %>


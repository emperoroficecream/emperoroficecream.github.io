# Test Katex

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.10.0-alpha/dist/katex.min.css" integrity="sha384-BTL0nVi8DnMrNdMQZG1Ww6yasK9ZGnUxL1ZWukXQ7fygA1py52yPp9W4wrR00VML" crossorigin="anonymous">
<script src="https://cdn.jsdelivr.net/npm/katex@0.10.0-alpha/dist/katex.min.js" integrity="sha384-y6SGsNt7yZECc4Pf86XmQhC4hG2wxL6Upkt9N1efhFxfh6wlxBH0mJiTE8XYclC1" crossorigin="anonymous"></script>


<span id="math"></span>

<script>
  var span = document.getElementById('math');
  katex.render("c = \\pm\\sqrt{a^2 + b^2}", span);
</script>
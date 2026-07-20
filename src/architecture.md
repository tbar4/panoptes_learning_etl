<iframe id="archframe" src="arch-embed.html" title="Panoptes ETL — types & data flow" scrolling="no" loading="eager" style="width:100%;border:0;display:block;min-height:70vh"></iframe>

<script>
(function () {
  var f = document.getElementById('archframe');
  function fit() {
    try {
      var d = f.contentDocument;
      if (d && d.documentElement) f.style.height = (d.documentElement.scrollHeight + 24) + 'px';
    } catch (e) {}
  }
  f.addEventListener('load', function () {
    fit();
    [200, 600, 1200, 2200, 3500].forEach(function (t) { setTimeout(fit, t); });
    try { if (window.ResizeObserver) new ResizeObserver(fit).observe(f.contentDocument.body); } catch (e) {}
  });
})();
</script>

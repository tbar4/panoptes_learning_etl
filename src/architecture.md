<iframe id="archframe" src="arch-embed.html" title="Panoptes ETL — types & data flow" scrolling="no" loading="eager" style="width:100%;border:0;display:block;min-height:70vh"></iframe>

<script>
(function () {
  var f = document.getElementById('archframe');
  function fit() {
    try {
      var w = f.contentDocument && f.contentDocument.querySelector('.wrap');
      if (w) f.style.height = (Math.ceil(w.getBoundingClientRect().height) + 40) + 'px';
    } catch (e) {}
  }
  f.addEventListener('load', function () {
    fit();
    [150, 500, 1000, 2000, 3500].forEach(function (t) { setTimeout(fit, t); });
    try {
      var w = f.contentDocument.querySelector('.wrap');
      if (window.ResizeObserver && w) new ResizeObserver(fit).observe(w);
    } catch (e) {}
  });
})();
</script>

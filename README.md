
function readImage(file) {
  return new Promise(function (resolve, reject) {
    var fr = new FileReader();

    fr.onerror = function () {
      reject(new Error('read'));
    };

    fr.onload = function () {
      var im = new Image();

      im.onerror = function () {
        reject(new Error('decode'));
      };

      im.onload = function () {
        var max = 720;
        var k = Math.min(1, max / Math.max(im.width, im.height));

        var c = document.createElement('canvas');
        c.width = Math.max(1, Math.round(im.width * k));
        c.height = Math.max(1, Math.round(im.height * k));

        var x = c.getContext('2d');
        x.fillStyle = '#ffffff';
        x.fillRect(0, 0, c.width, c.height);
        x.drawImage(im, 0, 0, c.width, c.height);

        resolve(c.toDataURL('image/jpeg', 0.8));
      };

      im.src = fr.result;
    };

    fr.readAsDataURL(file);
  });
}

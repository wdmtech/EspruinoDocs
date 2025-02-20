<!--- Copyright (c) 2020 Gordon Williams, Pur3 Ltd. See the file LICENSE for copying permission. -->
Font Converter
========================

<span style="color:red">:warning: **Please view the correctly rendered version of this page at https://www.espruino.com/Font+Converter. Links, lists, videos, search, and other features will not work correctly when viewed on GitHub** :warning:</span>

* KEYWORDS: Font,Create,Convert
* USES: Graphics

This page helps you to convert a font into JavaScript that can be used
with Espruino's [Graphics](/Graphics) library.

How it works:

* Go to [https://fonts.google.com/](https://fonts.google.com/), find a font
* Click `Select This Style`
* Copy the `&lt;link href=...` line
* Paste it into the text box below (or use a link to a '.otf' font)
* Check the options
* Click 'Go'
* Copy and paste the font code out

<form id="fontForm">
<input id="fontLink" type="text" value="<link href=&quot;https://fonts.googleapis.com/css2?family=Open+Sans+Condensed:wght@700&display=swap&quot; rel=&quot;stylesheet&quot;>" size="80"></input><br/>
Size : <input type="range" min="4" max="90" value="16" class="slider" style="width:500px" id="fontSize"><span id="fontSizeText">16</span><br/>
BPP : <select id="fontBPP">
  <option value="1" selected>1 bpp (black & white)</option>
  <option value="2">2 bpp</option>
  <option value="4">4 bpp</option>
</select><br/>
Range : <select id="fontRange">
  <option value="ASCII">ASCII (32-127)</option>
  <option value="ASCIICAPS" selected>ASCII capitals (32-127)</option>
  <option value="ISO8859-1">ISO8859-1 / ISO Latin (32-255)</option>
  <option value="Numeric">Numeric (46-58)</option>
</select><br/>
</form>
<button id="calculateFont">Go!</button><br/>

<span style="display:none;" id="fontTest" >This is a test of the font</span><br/>
<canvas width="256" height="256" id="fontcanvas" style="display:none"></canvas>
<textarea id="result" style="width:100%;display:none" rows="16"></textarea>
<canvas id="fontPreview" style="display:none;border:1px solid black;width:100%;image-rendering: pixelated;"></canvas>
<script>
var fontRanges = {
 "ASCII" : {min:32, max:127},
 "ASCIICAPS" : {min:32, max:93},
 "ISO8859-1" : {min:32, max:255},
 "Numeric" : {min:46, max:58},
};
var cssNode;

function createFont(fontName, fontHeight, BPP, charMin, charMax) {
  var canvas = document.getElementById("fontcanvas");
  var ctx = canvas.getContext("2d");
  ctx.font = fontHeight+"px "+fontName;

  function drawChSimple(ch, ox, oy) {
    var xPos = 0;
    ctx.fillStyle = "black";
    ctx.fillRect(xPos,0,fontHeight*2,fontHeight);
    ctx.fillStyle = "white";  
    ctx.fillText(ch, xPos+ox, fontHeight+oy-2);  
    var chWidth = Math.round(ctx.measureText(ch).width);
    var img = { width:0, height:fontHeight, data:[] };
    if (chWidth)
      img = ctx.getImageData(xPos,0,chWidth,fontHeight);
    return img; // data/width/height
  }

  // This one draws the same character at different offsets to try and get the clearest image
  // clearest image = most bright pixels
  function drawCh(ch) {
    var adjust = [{x:0,y:0},{x:-0.5,y:0},{x:0,y:-0.5},{x:-0.5,y:-0.5}];
    var bestPixelCnt = -1, bestImg;
    adjust.forEach(a=>{
      var img = drawChSimple(ch, a.x, a.y);
      var brightPixels = 0;
      for (var i=0;i<img.data.length;i+=4)
        if (img.data[i]>200)
          brightPixels++;
      if (brightPixels > bestPixelCnt) {
        bestPixelCnt = brightPixels;
        bestImg = img;
      }
    });
    return bestImg;
  }

  var preview = document.getElementById("fontPreview");
  preview.style.display = "inherit";
  var prevCtx = preview.getContext("2d");
  preview.width = fontHeight*16;
  preview.height = fontHeight*16;
  prevCtx.width = fontHeight*16;
  prevCtx.height = fontHeight*16;
  var prevImg = prevCtx.createImageData(fontHeight,fontHeight);

  var fontData = [];
  var bitData = 0, bitCount = 0;
  var fontWidths = [];
  var maxCol = 0, maxP = 0;
  var minY = 10000, maxY = -1;
  for (var ch=charMin;ch<=charMax;ch++) {
    var img = drawCh(String.fromCharCode(ch));
    fontWidths.push(img.width);
    prevImg.data.fill(255);
    for (var x=0;x<img.width;x++) {
      var s = "";
      for (var y=0;y<img.height;y++) {
        var idx = (x + y*img.width)*4;
        // get greyscale
        var c = (img.data[idx]+img.data[idx+1]+img.data[idx+2]) / 3;
        if (c>maxCol)maxCol=c;          
        // shift down to BPP with rounding
        c = (c + (127>>BPP)) >> (8-BPP);
        if (c>=(1<<BPP)) c = (1<<BPP)-1;
        // debug
        if (c>maxP) maxP=c;
        if (c) {
          if (y > maxY) maxY = y;
          if (y < minY) minY = y;
        }
        //if (ch=="X".charCodeAt()) console.log(x,y,c);
        s += " ,/#"[c>>(BPP-2)];
        var n = (x+(y*fontHeight))*4;
        var prevCol = 255 - (c << (8-BPP));
        prevImg.data[n] = prevImg.data[n+1] = prevImg.data[n+2] = prevCol;
        // add bit data
        bitData = (bitData<<BPP) | c;
        bitCount += BPP;
        if (bitCount>=8) {
          fontData.push(bitData);
          bitData = 0;
          bitCount = 0;
        }
      }
      //console.log(s);
    }
    prevCtx.putImageData( prevImg, (ch&15)*fontHeight, (ch>>4)*fontHeight );     
  }

  //console.log("Max color value = "+maxCol+", in bpp "+maxP);
  // if all fonts are the same width...
  var fixedWidth = fontWidths.every(w=>w==fontWidths[0]);

  var result = document.getElementById("result");
  result.style.display = "inherit";
  result.innerHTML = `
// Actual height ${maxY+1-minY} (${maxY} - ${minY})
${fixedWidth?"":`var widths = atob("${btoa(String.fromCharCode.apply(null,fontWidths))}");`}
var font = atob("${btoa(String.fromCharCode.apply(null,fontData))}");
var scale = 1; // size multiplier for this font
g.setFontCustom(font, ${charMin}, ${fixedWidth?fontWidths[0]:"widths"}, ${fontHeight}+(scale<<8)+(${BPP}<<16));
  `.trim();
}

function loadFontAndCalculate() {
  fontLink = document.getElementById('fontLink').value.trim();
  fontName = "Sans Serif";
  if (fontLink!="") {
    if (fontLink.startsWith("http")) {
      console.log("fontLink: Found bare URL");
    } else if (fontLink.startsWith("<")) {
      console.log("fontLink: Found <link...");
      var m = fontLink.match(/href="([^"]+)"/);
      if (m!==null) {
        console.log("fontLink: Found CSS Link");
        fontLink = m[1];
      } else {
        alert("Malformed Font link");
        return;
      }
    } else {
      console.log("fontLink: Assuming it's a font name");
      fontName = fontLink;
      fontLink = "";
    }
    if (fontLink) {
      fontName = undefined;
      var m;
      m = fontLink.match(/family=([%\w+]+)/);
      if (m!==null)
        fontName = decodeURI(m[1].replace(/\+/g," "));
      if (fontName===undefined) {
        m = fontLink.match(/([^/]*)\.otf/);
        if (m!==null)
          fontName = decodeURI(m[1]);      
      }
      if (fontName===undefined) {
        alert("Unable to work out font family from link");
        return;
      }
      if (fontLink.includes("#"))
        fontLink = fontLink.substr(0,fontLink.indexOf("#"));
    }
  }
  console.log("URL:" + (fontLink?fontLink:"[none]"));  
  console.log("Family:" + fontName);  
  var fontHeight = parseInt(document.getElementById('fontSize').value);
  var fontBPP = parseInt(document.getElementById("fontBPP").value);
  var fontRangeName =  document.getElementById("fontRange").value;
  var fontRange = fontRanges[fontRangeName];
  if (!fontRange) throw new Error("Unknown font range");

  document.getElementById('fontTest').style = `font-family: '${fontName}';font-size: ${fontHeight}px`;


  function callback() {
    createFont(fontName, fontHeight, fontBPP, fontRange.min, fontRange.max);
  }

  if (fontLink=="" || (cssNode && cssNode.href == fontLink)) {
    console.log("Font already loaded");
    return callback();
  }
  if (cssNode) cssNode.remove();
  if (fontLink.match(/\.otf([?#].*)?/)) {
    cssNode = document.createElement("style");
    cssNode.innerText = '@font-face { font-family: '+fontName+'; src: url('+JSON.stringify(fontLink)+') format("opentype"); }';
    cssNode.href = fontLink;
  } else {
    // assume CSS
    cssNode = document.createElement("link");
    cssNode.rel = "stylesheet";
    cssNode.type = "text/css";
    cssNode.href = fontLink;
  }
  var head = document.getElementsByTagName("head")[0];
  head.appendChild(cssNode);

  console.log("Waiting for font load");
  cssNode.onload = function() {
    setTimeout(function() {
      console.log("Font loaded");
      callback();
    }, 100);
  };
}
document.getElementById("calculateFont").addEventListener('click',function(e) {
  e.preventDefault();
  loadFontAndCalculate();
});
document.getElementById('fontSize').addEventListener('mousemove',function() {
  document.getElementById('fontSizeText').innerHTML = document.getElementById('fontSize').value;
});
document.getElementById("fontForm").addEventListener('submit', function(e) {
  e.preventDefault();
  loadFontAndCalculate();
});

</script>

The below flows for rendering with the plantuml github app are as follows:

`GitHub Camo → wrapper (/proxy) → GitHub raw/API → PlantUML (/png) → wrapper → Camo → browser`

<!-- commenting out original plantuml site to eliminate caching
     leaving this in for future reference for myself

### plantuml repo via raw.github.com 

![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.github.com/plantuml/plantuml-server/master/src/main/webapp/resource/test/test2diagrams.txt)

`![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.github.com/plantuml/plantuml-server/master/src/main/webapp/resource/test/test2diagrams.txt)`

### plantuml repo via raw.githubusercontent.com
 
![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.githubusercontent.com/plantuml/plantuml-server/master/src/main/webapp/resource/test/test2diagrams.txt)

`![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.githubusercontent.com/plantuml/plantuml-server/master/src/main/webapp/resource/test/test2diagrams.txt)`
 
-->

---

### unauthorized repo via raw.github.com 
_should not render_

![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.github.com/joelparkerhenderson/plantuml-examples/refs/heads/master/doc/mind-map/mind-map.plantuml)

`![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.github.com/joelparkerhenderson/plantuml-examples/refs/heads/master/doc/mind-map/mind-map.plantuml)`

---

### unauthorized repo via raw.githubusercontent.com
_should not render_
 
![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.githubusercontent.com/joelparkerhenderson/plantuml-examples/refs/heads/master/doc/mind-map/mind-map.plantuml)

`![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.githubusercontent.com/joelparkerhenderson/plantuml-examples/refs/heads/master/doc/mind-map/mind-map.plantuml)`

---

### this repo via raw.github.com
_should not render_

![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.github.com/cdxsystems/public-test-repo/main/puml/mind-map.puml)

`![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.github.com/cdxsystems/public-test-repo/main/puml/mind-map.puml)`

---

### this repo via raw.githubusercontent.com
__should render__

![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.githubusercontent.com/cdxsystems/public-test-repo/main/puml/mind-map.puml)

`![plantuml test image](http://cgxhbnr1bwwk.y2r4c3lzdgvtcy5jb20k.com/proxy?cache=no&src=https://raw.githubusercontent.com/cdxsystems/public-test-repo/main/puml/mind-map.puml)`

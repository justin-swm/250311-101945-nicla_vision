#### Transfer Learning Modelle
Für die erkennung wurde sich für die Bilderknnung entschieden Edge Impulse bietet Hiefür die option des [Transferlarnings](https://docs.edgeimpulse.com/docs/edge-impulse-studio/learning-blocks/transfer-learning-images) hier bei wird ein bestehendes bereits Trainiertes modell verwendet und auf einen neuen aber ähnlichen anwendungszweck trainiert. Ein großer vorteil daran ist das so wenniger daten und weniger Trainigszeit benötigt werden. Hierbei bedint sich Transferlearning der möglichkeit ein bestehenden Feature Extraktor beizubahalten aber das Netz zu Trainieren aber auf eine neue Ausgabe zu Trainieren also unsere zu erkennen Klassen dies geschiet in den oberen layer des Netzes [vgl. Transfer Leraning](https://datascientest.com/de/transfer-learning) deswegen werden diese weggeschnitten und durch neue layer ersetzt. Edge impulse bietet hier bereits eine vorbereitete Version diese wird im Folgenden kurz beschrieben. 

So werden in diesem Projekt die oberen 3 Layer entfert:
```Python
last_layer_index = -3
model.add(Model(inputs=base_model.inputs, outputs=base_model.layers[last_layer_index].output))****
```
Satdessen wird ein eigener Klassifikationskopf erstellt:
```Python
model.add(Reshape((-1, model.layers[-1].output.shape[3])))
model.add(Dense(8, activation='relu'))
model.add(Dropout(0.1))
model.add(Flatten())
model.add(Dense(classes, activation='softmax'))
```

Zusätzlich wird nach dem training des neuen Klassifikationskopf auch der Feature Extraktor noch einmal optimiert.
```Python
model = ei_tensorflow.training.load_best_model(BEST_MODEL_PATH)
# Allow the entire base model to be trained
model.trainable = True
# Freeze all the layers before the 'fine_tune_from' layer
for layer in model.layers[:fine_tune_from]:
    layer.trainable = False
``` 

Bei den oben stehenden code handelt es sich um auschnitte des Codes den Edge impulse im [Experten modus](https://docs.edgeimpulse.com/docs/edge-impulse-studio/learning-blocks/transfer-learning-images#expert-mode) anzeigt. Er beziht sich auf das MobileNet V2 modell bei dem V1 modell sieht es anders aus.  Hier gibt es thortisch auch die möglichkeit das modell weiter zu Optimieren dies wurde im Projekt nicht genutzt da die auswahl an verschieden Modellen vorerst groß genug war hier gibt es aber für erfahren nutzer sichelrich die möglichkeit für weitere anpassungen.

##### Überblick über verschiedene Modelle
Im Folgenden abschnitt sollen nun die in Edge Impulse Zur verfügung Stehenden Modele genauer betrachtet werden. Edge Impusle bietet aktuell 3 [Architekuren](https://docs.edgeimpulse.com/docs/edge-impulse-studio/learning-blocks/transfer-learning-images#expert-mode) für Transferleqarning mit bildern an. Dabei wird jedes modell dann mit verschiedenen Alpha werten angeboten. Dieser wert auch  als [width multiplier](https://viso.ai/deep-learning/mobilenet-efficient-deep-learning-for-mobile-vision/#mobilenet-architecture) beinflusst die Anzahl der kanäle je Layer. Damit lässt sich das modell verkleinern und die geschwindigkeit erhöhen. Allerdings mit einfluss auf die genaigkeit genaueres dazu im Abschnitt [Ergebnisse](#ergebnisse).

Zur Auswahl Stehen die Modelle [MobileNetV1](https://docs.edgeimpulse.com/docs/edge-impulse-studio/learning-blocks/transfer-learning-images#mobilenetv1-1), [MobileNetV2](https://docs.edgeimpulse.com/docs/edge-impulse-studio/learning-blocks/transfer-learning-images#mobilenetv2-1) und [EfficientNet](https://docs.edgeimpulse.com/docs/edge-impulse-studio/learning-blocks/transfer-learning-images#efficientnet-1). Genau betrachtet werden allerdings nur die beiden Modelle der MobileNet famielie da die EfficientNet modelle mindestens 4mb an RAM Speicher in anspruch nehmen und damit deutlich zu groß für den [Arduino Nicla Vision](#nicla-vision) sind. 

Die MobileNet V1 Modelle wurden von einem Team von Forschern von Googel im Jahr 2017 [veröfentlicht](https://arxiv.org/abs/1704.04861).  Heirbei handelt es sich um ein Klasse von Modellen speziel für Mobile und Embeded Anwendungen im bereich des Sehens. Dabei wurde die größe der modelle dratisch verkleinert bei guter leistung im vergelich zu anderen Modellen [vgl.MobileNet 4](https://arxiv.org/pdf/1704.04861).   Das Modell verwendet [Depthwise Separable Convolutions](https://viso.ai/deep-learning/mobilenet-efficient-deep-learning-for-mobile-vision/#depthwise-convolution-and-pointwise-convolution) annstat einer Klassischen Faltung über alle kanäle aufeinmal  wird dieser Prozzes auf zwei Schritte aufgeteilt [vgl. Depthwise Convolution and Pointwise Convolution](https://viso.ai/deep-learning/mobilenet-efficient-deep-learning-for-mobile-vision/#depthwise-convolution-and-pointwise-convolution). Hierbei wird zunächst ein einzelner Filter für jeden Kanal verwendte (Depthwise convolution). Anschliesen werden für den Outputs dann mit einem 1x1 filter die kombiniert dies geschiet für jedes Pixel (Pointwise Convolution). Wichtig ist in diesem schritt das der 1x1 filter alle Kanäle pixel für pixel berücksichtig. Daraus entsteht eine neue Feature Map wie bei der Klassischen Faltung eben aufgetilt in 2 schritten [vgl. Depthwise Separable Convolutions](https://viso.ai/deep-learning/mobilenet-efficient-deep-learning-for-mobile-vision/#depthwise-separable-convolutions). 
![Depthwise Separable Convolutions](img\DepthwiseSeparableConvolutions.jpg)
[sorce](https://viso.ai/wp-content/uploads/2024/04/depthwise.jpg)
Durch dieses aufteilung auf 2 schritte reduziert sich der rechenaufwand deutlich bei ähnlich guten ergbnissen [vgl. MobileNets 3.1](https://arxiv.org/pdf/1704.04861). Eine genauere auführung hierzu findet sich in dem [Paper](https://arxiv.org/pdf/1704.04861). Zur besseren grafischen veranschaulichung gibt es auch dieses [Video](https://www.youtube.com/watch?v=vVaRhZXovbw)

Die Architekur bassiert zu großen teilen aus Depthwise Separable Convolutions wobei einige layer wie der erste davon abweichen beim ersten layer handelt es sich um eine  "Full convolution" [vgl 3.2. Network Structure and Training](https://arxiv.org/pdf/1704.04861). 
In der nachfolgenden grafik ist die architekur dargestelt.
![MobileNet Body Architecture](img\mobileNet-architektur.png)
[sorce](https://viso.ai/deep-learning/mobilenet-efficient-deep-learning-for-mobile-vision/#mobilenet-architecture)

Die Entwickler geben zu dem an das 95% der rechenzeit für 1x1 faltungen verwendet. diese machen zudem 75% der Paramter aus [vgl. 3.2 Network Structure and Training](https://arxiv.org/pdf/1704.04861). 

![Resource Per Layer Type](img\resorce-perLayer.png)
[source](https://arxiv.org/pdf/1704.04861)

MobileNet Modelle sind zwar schon klein aber noch nicht klein genug für die Nutzung auf Edge Devices hiefür bietet MobileNet das bereits erwähnten Width Multiplier (alpha) dieser wert reduziert die anzahl der in und output kanäle [vgl 3.3. Width Multiplier: Thinner Models](https://arxiv.org/pdf/1704.04861). Das bedeutet das wenn Standertmässig 32 Input Kanäle vorhanden sind macht ein alpha = 0.5 daraus 16 Kanäle. Mit dem  sinken der rechenaufwand und anzahl der parameter etwa im verhätnis 
alpha^2. 

Der zweite anpassbare wert ist der Resolution Multiplier ρ dieser bestimmt die auflössung des input images, dardurch reduziert sich auch die auflösung aller weiteren layer [vgl. 3.4. Resolution Multiplier: Reduced Representa-
tion](https://arxiv.org/pdf/1704.04861). Dies geschieht inplizit durch das festlegen der input resolution. Der standert wäre 224 weitere auflösungen und die daraus folgende parammter größe kann man der grafik entnehmen.
![](img\MobileNet_Resolution.png)

Edge impulse biettet uns nun die möglichkeit zu wählen zwischen 3 verschieden Alphas 0.25 , 0.2 und 0.1.   Die auflösung ist immer auf 96x96 festgelgt. Beide werte sind unter dem was in dem paper beschreiben wird. Dies hängt mit der geringen leistungsfähigkeit der edge Devices Zusammen. Dies führt aber auch zu einer sehr regringen ram bedarf von nur etwas über 100kb. 

MobileNet V2 wurde 2018 als Weiterentwicklung von MobileNet V1 vorgestellt. Es basiert weiterhin auf Depthwise Separable Convolutions, nutzt jedoch zusätzlich sogenannte Inverted Residual Blocks mit Bottleneck-Struktur. Dadurch wird die Anzahl der Parameter und die Rechenlast weiter reduziert – bei vergleichbarer Genauigkeit [vgl. Keras](https://keras.io/api/applications/mobilenet/#mobilenetv2-function).
Ein Inverted Residual Block besteht vereinfacht aus drei Schritten:
(1) Expansion: Die Anzahl der Kanäle wird zunächst durch eine 1×1-Convolution erhöht, um dem Modell mehr Kapazität zur Merkmalsextraktion zu geben.
(2) Depthwise Convolution: Danach wird eine 3×3-Depthwise Convolution pro Kanal separat angewendet, was räumliche Filterung mit sehr geringem Rechenaufwand ermöglicht.
(3) Lineare Projektion (Bottleneck): Zum Schluss wird die Kanalanzahl durch eine weitere 1×1-Convolution wieder reduziert, wobei bewusst auf eine Aktivierungsfunktion verzichtet wird, um Informationsverluste zu vermeiden.
Nur wenn Eingabe- und Ausgabedimension gleich sind, wird eine Residual-Verbindung (Skip-Connection) hinzugefügt, die das ursprüngliche Signal direkt auf den Ausgang addiert [vgl. 3. Preliminaries, discussion and intuition](https://arxiv.org/pdf/1801.04381). Es handelt sich hier also um eine Erweiterung der Depthwise Separable Convolutions wobei die Expansion zusätzlich eingebaut wird.

Der Aufbau eines Inverted Residual Blocks:
![](img\InvertedResidualBlocks.webp)
[source](https://miro.medium.com/v2/resize:fit:640/format:webp/1*5Jdh_PDTXp0uhF8c79TEsQ.png)

Hier ist die Expansion gut zu erkenen. Der Schwarze Pfeil stellt die skip konektion da dieser verbindet den  Input eines Layers direkt mit dem Output. Dies wird für stabilere Trainings Tiefer netze benötigt [vgl. Inverted Residuals](https://medium.com/data-science/mobilenetv2-inverted-residuals-and-linear-bottlenecks-8a4362f4ffd5). 

Das [Paper](https://arxiv.org/abs/1801.04381) zum MobileNet V2 beschreibt auch die größe und den rechenaufwand im vergleich zum V1 model.
![Size](img\V2_size.png)
Zu erkennen ist hier schnell das sowohl die Geschwindigkeit höher und die größe kleiner geworden ist. Dabeis ist das ergbniss besser geworden.
Edge Impulse bietet auch hier wieder eine [Auswahl an modellen](https://docs.edgeimpulse.com/docs/edge-impulse-studio/learning-blocks/transfer-learning-images#mobilenetv2-1).
Hierbei gibt es 2 auflösungen zur auswahl wie beim V1 Modell 96x96 was einem faktor ρ von etwa 0,43 entspricht. Für das alpha gibt es drei varianten 0.05 ,0.1 und 0.35 alle drei modelle sind mit circa 260-300 kb Rom nutzbar [(einschränkungen beachten)](#speicherprobleme-und-deren-behebung).
Die Modelle mit der auflösung von 160x160 sind mit größeren alphas verfügbar 0.35, 0.5, 0.75 und 1.0  diese modell  sind mit  680kb bis 1.3 mb Ram bedarf zu groß für die nutzung auf dem Nicla Vision.

 Zu bachten ist an der Stelle allerdings das die in Edge impulse genutzten Modelle kleinre Alphas und Auflösungen nutzen müssen und auch noch Quantisiert werden müssen. Genaureres zur RAM nutzung in dem Abschnitt [Speicherprobleme](#speicherprobleme-und-deren-behebung).  Welche Ergbnisse mit den beiden Modellen ereicht werde konnten wird im Abschnitt [Ergebnisse](#ergebnisse) beschrieben. 

Dies soll als Übersicht genügen das Thema ist nartürlich duelich umfangreicher und kompelexer als hier beschriben könnten aber  interesssant für zukünftige Projekte sein. 

Die Paper zum [V1](https://arxiv.org/abs/1704.04861) und [V2](https://arxiv.org/abs/1801.04381) Modell geben hier eine gute Anlaufstelle. 
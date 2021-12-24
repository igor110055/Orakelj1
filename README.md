# Orakelj1

Recimo, da izdelujemo Dapp in hočemo uporabnikom aplikacije omogočiti,
da dvignejo Eth, v določeni količini USD.
Da zadostimo tej zahtevi mora naša pametna pogodba poznati vrednost Eth v USD. 

JavaScript aplikacija lahko preprosto prenese tako vrsto informacije z Binance public API 
(ali kakšnega durgega servica, ki javno daje pricefeede). Ampak pametna pogodba nima 
direktnega dostopa do kakršnihkoli informacij z zunanjega sveta. Namesto tega se za 
dostop do podatkov zanaša na orakelj.

SLIKA

# Postavljanje ogrodja

V terminalu se postavimo v mapo našega projekta. Nato naredimo mapo EthPriceOracle
in se postavimo v njo. 

Zaženemo ukaz:

```
npm init -y
```

Nato pa uvozimo naslednje pakete: truffle, openzeppelin-solidity, loom-js, loom-truffle-provider, bn.js, and axios.

(pri uvozu loom-js je morda potrebno naložiti verzijo loom-js@1.75.2)

Zdaj, ko imamo vse pripravljeno, naredimo dve mapi oracle in caller ter v njih zaženemo truffle.
To storimo tako, da zaženemo spodnja ukaza:

```
mkdir oracle && cd oracle && npx truffle init && cd ..

mkdir caller && cd caller && npx truffle init && cd ..
```

Naša mapa naj bi imela spodnjo obliko:
```
.
├── caller
│   ├── contracts
│   ├── migrations
│   ├── test
│   └── truffle-config.js
├── oracle
│   ├── contracts
│   ├── migrations
│   ├── test
│   └── truffle-config.js
└── package.json

```


# ci-workflows

Közös, újrahasznosítható GitHub Actions workflow-k (`on: workflow_call`), amiket
a többi repó `uses: harold4/ci-workflows/.github/workflows/<név>.yml@v1`-gyel hív.

## FONTOS: a repó PUBLIKUS

- Secret soha nem kerül bele — a hívó adja át `secrets:`-szel.
- README/példák csak fiktív neveket használnak (`my-app`, `MY_TOKEN`); privát
  projekt neve, secret-neve, belső útvonala nem szerepelhet.
- Push előtt a diffet ebből a szempontból is át kell nézni.

## Tech stack

GitHub Actions YAML. Nincs build, nincs futtatható kód.

## Fájlstruktúra

```
.github/workflows/docker.yml   Docker image build + push GHCR-re + régi buildek takarítása
README.md                      inputok, secretek, outputok, hívó példák (angol)
```

## Verziózás

- A hívók főverzió-taggel hivatkoznak (`@v1`).
- Kompatibilis változás: commit, push, majd a `v1` tag átállítása:
  `git tag -f v1 && git push -f origin v1`.
- Törő változás (input átnevezése/törlése, default megváltozása): új `v2` tag,
  a hívók egyenként állnak át.

## Ellenőrzés

Nincs helyi teszt; a változást egy hívó projekt workflow-ja validálja
(pl. `workflow_dispatch`-csel indítva). Ha elérhető, `actionlint`.

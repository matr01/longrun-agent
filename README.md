# longrun-agent

Opakovaně spouštěný agent na GitHub Actions, jehož běhy na sebe navazují
a postupně tvoří jeden produkt (nejdřív dokumenty v Markdownu, později i kód).
Vychází z principu [dsh-monitor](https://github.com/matr01/dsh-monitor),
ale běhy nejsou nezávislé: stav, plán a výsledky se ukládají do repozitáře.

## Stav projektu

Fáze návrhu, zatím nic není implementováno.
Cíle, dílčí problémy a předběžná řešení: [docs/overview.md](docs/overview.md).

## Princip

- plán rozdělený na malé úkoly, každý běh řeší jeden
- role worker / planner / reviewer
- zásahy zvenčí zvažovány přes GitHub issues
- všechno v gitu, tedy kontrola přes diff

[[Varios]]

Apuntes personales de aprendizaje sobre el mundo cripto, con foco en Solana (no es un proyecto de código, es documentación de estudio para meter las manos con criterio).

## El panorama en 4 capas

```
Base:        Wallet  +  Solana (blockchain)
                 └──────┬──────┘
Activo:            Tokens (SOL / SPL)
                 ┌───────┼───────┐
Actividad:     DeFi   Staking   NFTs
                 └───────┼───────┘
Recompensa:          Airdrops
```

- **Wallet**: par de llaves criptográficas. La pública es tu dirección; la privada (o la frase semilla de 12-24 palabras) firma transacciones. **Quien tenga esa frase controla los fondos — no hay "recuperar contraseña".** Nunca guardarla en foto, nube o texto plano.
- **Solana**: la blockchain donde vive todo esto. Rápida (~400ms/bloque) y barata (fracciones de centavo por transacción) comparada con Ethereum.
- **Tokens**: SOL es la moneda nativa (paga el gas). Los tokens SPL son todo lo demás construido sobre Solana (USDC, tokens de proyectos, etc.).
- **DeFi / Staking / NFTs**: las formas de *usar* los tokens — swaps y préstamos, bloquear SOL con un validador para ganar interés (~5-7% anual), o coleccionar NFTs.
- **Airdrops**: no es algo que se "usa", es un efecto secundario de haber usado las capas de abajo. Un proyecto sin token todavía reparte una asignación gratuita a quienes usaron su protocolo tempranamente.

## Glosario rápido

- **Farmear un airdrop**: generar actividad on-chain a propósito (swaps, volumen, staking, quests) para calificar a una futura distribución de tokens, aunque no te interese el producto en sí.
- **SolanaHub** ([solanahub.app](https://solanahub.app/), [repo](https://github.com/Avaulto/SolanaHub)): dApp "todo en uno" para el ecosistema Solana — wallets, balances SPL, staking con validadores, DeFi, utilidades NFT y estado de la red, todo en un solo lugar.
- **SPL**: el estándar de tokens de Solana (equivalente a ERC-20 en Ethereum).
- **Sybil attack** (en contexto de airdrops): usar múltiples wallets para simular varios usuarios distintos y multiplicar la recompensa; los proyectos filtran esto activamente.

## Riesgos y cómo empezar con seguridad

- Los sitios/apps falsas que imitan hubs o wallets legítimos son el vector de estafa más común — verificar siempre el dominio oficial antes de conectar una wallet o firmar algo.
- Nunca hay que ingresar la frase semilla en ningún sitio web, formulario o "verificación" — ninguna app legítima la pide.
- Orden recomendado para arrancar:
  1. Crear una wallet y guardar la frase semilla offline.
  2. Comprar algo de SOL en un exchange y transferirlo a la wallet propia.
  3. Recién ahí explorar DeFi/staking, con montos pequeños que se puedan perder sin dolor.
- Muchos proyectos detectan y excluyen wallets con patrones de farming artificial — no hay garantía de recompensa por "farmear".

## Cómo crear una wallet (Phantom) paso a paso

Phantom es la wallet más usada en Solana. Dominio oficial de descarga: **phantom.com/download** — nunca instalar desde un link de X/Twitter, Google Ads o un DM, aunque parezca legítimo.

1. **Instalar la extensión** en el navegador (Chrome/Brave/Firefox) desde phantom.com/download. Verificar que el publisher en la Chrome Web Store sea "Phantom" antes de aceptar.
2. **Elegir "Create a Recovery Phrase Wallet"** (no login con Google/Apple) — este método es el que te hace dueño real de la llave privada, que es el punto central de entender cripto. El login social delega parte de la custodia a Phantom/Google.
3. **Crear una contraseña local** — protege el acceso a la extensión en este dispositivo, no es la llave real de los fondos.
4. **Anotar la frase de recuperación de 12 palabras a mano, en papel, en orden.** Nunca en foto, captura de pantalla o nube. Guardarla en un lugar físico seguro.
5. **Confirmar la frase** — Phantom pide reingresar algunas palabras para verificar que se copió bien.
6. Listo: ya existe una dirección pública (ej. `7xKX...`) para recibir fondos.

**Antes de fondear con dinero real:**
- Nunca vas a escribir esa frase de 12 palabras en ningún sitio web, ni dársela a "soporte" de nadie — ninguna entidad legítima la pide jamás.
- Al primer envío de fondos, probar primero con un monto pequeño antes de mover todo.

Fuente oficial: [Create a new Phantom wallet with a recovery phrase](https://help.phantom.com/hc/en-us/articles/45135465489555-Create-a-new-Phantom-wallet-with-a-recovery-phrase)

## Cómo comprar SOL y transferirlo a la wallet

Para Chile, **Buda.com** es la opción más simple: exchange chileno, CLP directo por transferencia bancaria, sin pasar por otra moneda. Compra/depósito toma 10-30 min; retiro a banco (si algún día vendes) toma 24-48h.

1. **Crear cuenta y verificar identidad (KYC)** — obligatorio en cualquier exchange regulado: cédula + selfie.
2. **Depositar CLP** por transferencia bancaria.
3. **Comprar SOL** — fee típico 0.8-1.0%.
4. **Copiar la dirección pública de Phantom** (ícono "Copy Address" junto al nombre de la cuenta) — nunca transcribirla a mano, un solo carácter mal copiado manda los fondos a una dirección inexistente o de otra persona, sin forma de revertirlo.
5. En Buda.com ir a **Retirar → SOL**, pegar la dirección.
6. **Primero un monto de prueba pequeño** (unos pocos dólares) y confirmar que llega a Phantom antes de mover el resto.
7. Verificar en Phantom que el monto de prueba llegó, luego retirar el resto con confianza.

**Regla de oro:** las transacciones en blockchain son irreversibles. Comparar visualmente el inicio y el final de la dirección pegada contra la de Phantom antes de confirmar — hay malware que cambia direcciones copiadas al portapapeles.

Fuentes: [Blockchain.cl — Guía Solana 2026](https://www.blockchain.cl/solana-guia-completa/), [99Bitcoins — Cómo comprar Solana](https://99bitcoins.com/guides-and-tutorials/buy-solana-sol/)

# Discord Guild Tag Otomatik Rol Sistemi

Belirli bir guild tagine sahip üyelere otomatik rol veren ve tag kaybolduğunda rolü geri alan Discord bot sistemi. Büyük sunucuları düzenli tutmak artık çok kolay. 😏

---

## Sistem Nasıl Çalışıyor?

Bu sistem Discord'un **Raw Event** mekanizmasını kullanır ve `GUILD_MEMBER_UPDATE` eventini dinler. Üyelerin tag durumunu sürekli kontrol eder.  

**İşleyiş:**

1. **Üye tag aldıysa ve rolü yoksa:**  
   - Rol otomatik olarak eklenir.  
   - Log kanalına yeşil bir embed gönderilir.  

2. **Üye tagı kaybettiyse ve rolü varsa:**  
   - Rol otomatik olarak kaldırılır.  
   - Log kanalına kırmızı bir embed gönderilir.  

3. **Spam önleme:**  
   - Aynı kullanıcıya kısa sürede tekrar işlem yapılmaması için `debounceCache` kullanılır.

---

## Örnek Kod

```ts
import { WosskiClient, Event } from '@repo/client'
import { bold, inlineCode, Colors, EmbedBuilder, TextChannel, GatewayDispatchPayload, Events } from 'discord.js'

const debounceCache = new Map<string, number>()

const config = {
  guildID: '852890250312286218',
  log: '1433226826691514378',
  role: '1415371533483901162'
}

export default class RawEvent extends Event {
  constructor(client: WosskiClient) {
    super(client, { name: Events.Raw })
  }

  async run(client: WosskiClient, packet: GatewayDispatchPayload) {
    if (packet.t !== 'GUILD_MEMBER_UPDATE') return
    const data = packet.d
    if (data.guild_id !== config.guildID) return

    const guild = client.guilds.cache.get(config.guildID)
    if (!guild) return
    const member = guild.members.cache.get(data.user.id)
    if (!member || member.user.bot) return

    const last = debounceCache.get(member.id)
    if (last && Date.now() - last < 3000) return
    debounceCache.set(member.id, Date.now())

    const logChannel = client.channels.cache.get(config.log) as TextChannel | undefined

    const hasTag = data.user.primary_guild?.identity_guild_id === config.guildID
    const hasRole = member.roles.cache.has(config.role)

    if (hasTag && !hasRole) {
      await member.roles.add(config.role).catch(() => {})
      logChannel?.send({
        embeds: [
          new EmbedBuilder()
            .setColor(Colors.Green)
            .setDescription(
              `${member} [${inlineCode(member.id)}] ${bold(`@${member.guild.roles.cache.get(config.role)?.name}`)} verildi`
            )
        ]
      })
    } else if (!hasTag && hasRole) {
      await member.roles.remove(config.role).catch(() => {})
      logChannel?.send({
        embeds: [
          new EmbedBuilder()
            .setColor(Colors.Red)
            .setDescription(
              `${member} [${inlineCode(member.id)}] ${bold(`@${member.guild.roles.cache.get(config.role)?.name}`)} alındı`
            )
        ]
      })
    }
  }
}
```
> Kendi event handlerına göre düzenlemen gerekebilir.

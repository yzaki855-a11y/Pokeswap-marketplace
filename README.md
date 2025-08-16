import React, { useMemo, useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Badge } from "@/components/ui/badge";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { Sheet, SheetContent, SheetFooter, SheetHeader, SheetTitle } from "@/components/ui/sheet";
import { Label } from "@/components/ui/label";
import { Checkbox } from "@/components/ui/checkbox";
import { Switch } from "@/components/ui/switch";
import { Separator } from "@/components/ui/separator";
import { Slider } from "@/components/ui/slider";
import {
  Search,
  Plus,
  Filter,
  ShoppingCart,
  Globe2,
  Star,
  ShieldCheck,
  BadgePercent,
  Tag,
  Coins,
  SendHorizontal,
  TrendingUp,
} from "lucide-react";
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  Tooltip,
  ResponsiveContainer,
} from "recharts";

// --- Types
export type CardItem = {
  id: string;
  name: string;
  set: string;
  number: string;
  rarity: "Common" | "Uncommon" | "Rare" | "Ultra Rare" | "Secret Rare";
  condition: "NM" | "LP" | "MP" | "HP" | "DMG";
  grading?: "PSA" | "BGS" | "CGC" | "Raw";
  grade?: number;
  currency: "USD" | "EUR" | "AED" | "GBP" | "JPY";
  price: number; // list price in card.currency
  seller: string;
  country: string;
  image: string;
  history: { date: string; price: number }[]; // last prices in USD for simplicity
};

const DEMO: CardItem[] = [
  {
    id: "1",
    name: "Charizard VMAX (Rainbow)",
    set: "Sword & Shield—Darkness Ablaze",
    number: "SV107/SV122",
    rarity: "Secret Rare",
    condition: "NM",
    grading: "PSA",
    grade: 10,
    currency: "USD",
    price: 699,
    seller: "@KantoCards",
    country: "USA",
    image:
      "https://images.pokemontcg.io/swsh3/20_hires.png",
    history: [
      { date: "2025-03-01", price: 620 },
      { date: "2025-04-01", price: 640 },
      { date: "2025-05-01", price: 655 },
      { date: "2025-06-01", price: 670 },
      { date: "2025-07-01", price: 699 },
    ],
  },
  {
    id: "2",
    name: "Pikachu (Illustrator)",
    set: "Promo",
    number: "098/SM-P",
    rarity: "Ultra Rare",
    condition: "LP",
    grading: "BGS",
    grade: 9,
    currency: "JPY",
    price: 120000,
    seller: "@TokyoTCG",
    country: "Japan",
    image:
      "https://images.pokemontcg.io/sm35/40_hires.png",
    history: [
      { date: "2025-03-01", price: 850 },
      { date: "2025-04-01", price: 1100 },
      { date: "2025-05-01", price: 950 },
      { date: "2025-06-01", price: 1000 },
      { date: "2025-07-01", price: 1050 },
    ],
  },
  {
    id: "3",
    name: "Umbreon Gold Star",
    set: "POP Series 5",
    number: "17/17",
    rarity: "Secret Rare",
    condition: "MP",
    grading: "Raw",
    currency: "EUR",
    price: 450,
    seller: "@EeveeEmporium",
    country: "Germany",
    image:
      "https://images.pokemontcg.io/pokemongo/22_hires.png",
    history: [
      { date: "2025-03-01", price: 420 },
      { date: "2025-04-01", price: 410 },
      { date: "2025-05-01", price: 430 },
      { date: "2025-06-01", price: 440 },
      { date: "2025-07-01", price: 450 },
    ],
  },
];

const RATES: Record<string, number> = {
  USD: 1,
  EUR: 1.09,
  GBP: 1.27,
  AED: 0.27,
  JPY: 0.0068,
};

const rarityColors: Record<CardItem["rarity"], string> = {
  Common: "bg-gray-100 text-gray-700",
  Uncommon: "bg-teal-100 text-teal-700",
  Rare: "bg-blue-100 text-blue-700",
  "Ultra Rare": "bg-purple-100 text-purple-700",
  "Secret Rare": "bg-yellow-100 text-yellow-800",
};

function formatMoney(value: number, currency: string) {
  try {
    return new Intl.NumberFormat(undefined, {
      style: "currency",
      currency,
      maximumFractionDigits: 0,
    }).format(value);
  } catch {
    return `${value.toFixed(0)} ${currency}`;
  }
}

export default function PokeSwapApp() {
  // UI state
  const [query, setQuery] = useState("");
  const [currency, setCurrency] = useState<"USD" | "EUR" | "AED" | "GBP" | "JPY">("USD");
  const [rarity, setRarity] = useState<string>("all");
  const [conditions, setConditions] = useState<string[]>(["NM", "LP", "MP"]);
  const [gradedOnly, setGradedOnly] = useState(false);
  const [priceRange, setPriceRange] = useState<number[]>([0, 2000]);
  const [newListingOpen, setNewListingOpen] = useState(false);
  const [detail, setDetail] = useState<CardItem | null>(null);
  const [offerOpen, setOfferOpen] = useState(false);
  const [escrow, setEscrow] = useState(true);
  const [sort, setSort] = useState("recommended");

  // Derived
  const filtered = useMemo(() => {
    const q = query.toLowerCase();
    const minUSD = priceRange[0] * RATES[currency];
    const maxUSD = priceRange[1] * RATES[currency];
    return DEMO.filter((item) => {
      const matchesQuery =
        item.name.toLowerCase().includes(q) ||
        item.set.toLowerCase().includes(q) ||
        item.number.toLowerCase().includes(q);
      const matchesRarity = rarity === "all" || item.rarity === (rarity as any);
      const matchesCondition = conditions.includes(item.condition);
      const usd = item.price * (RATES[item.currency] || 1);
      const within = usd >= minUSD && usd <= maxUSD;
      const matchesGrade = gradedOnly ? item.grading && item.grading !== "Raw" : true;
      return matchesQuery && matchesRarity && matchesCondition && within && matchesGrade;
    }).sort((a, b) => {
      if (sort === "price-asc") return a.price * RATES[a.currency] - b.price * RATES[b.currency];
      if (sort === "price-desc") return b.price * RATES[b.currency] - a.price * RATES[a.currency];
      if (sort === "newest") return (b.history?.length || 0) - (a.history?.length || 0);
      return 0; // recommended (static)
    });
  }, [query, rarity, conditions, gradedOnly, priceRange, currency, sort]);

  return (
    <div className="min-h-screen w-full bg-gradient-to-b from-neutral-50 to-white">
      {/* Header */}
      <header className="sticky top-0 z-30 backdrop-blur bg-white/70 border-b">
        <div className="max-w-7xl mx-auto px-4 py-3 flex items-center gap-3">
          <Globe2 className="h-6 w-6" />
          <h1 className="text-xl sm:text-2xl font-bold">PokéSwap</h1>
          <Badge variant="secondary" className="ml-1">Global TCG Marketplace</Badge>
          <div className="ml-auto flex items-center gap-2">
            <Select value={currency} onValueChange={(v:any)=>setCurrency(v)}>
              <SelectTrigger className="w-[110px]"><SelectValue placeholder="USD" /></SelectTrigger>
              <SelectContent>
                {(["USD","EUR","AED","GBP","JPY"] as const).map(c=> (
                  <SelectItem key={c} value={c}>{c}</SelectItem>
                ))}
              </SelectContent>
            </Select>
            <Button variant="outline" size="sm" className="gap-2" onClick={()=>setNewListingOpen(true)}>
              <Plus className="h-4 w-4"/> List a Card
            </Button>
            <Button size="sm" className="gap-2">
              <ShoppingCart className="h-4 w-4"/> Offers
            </Button>
          </div>
        </div>
      </header>

      {/* Search & Filters */}
      <section className="max-w-7xl mx-auto px-4 py-4">
        <Card className="shadow-sm border-neutral-200">
          <CardContent className="p-4">
            <div className="grid grid-cols-1 md:grid-cols-12 gap-3 items-center">
              <div className="md:col-span-6 flex items-center gap-2">
                <Search className="h-5 w-5 text-neutral-500"/>
                <Input
                  value={query}
                  onChange={(e)=>setQuery(e.target.value)}
                  placeholder="Search by name, set, or number (e.g., 4/102 Base Set)"
                  className="w-full"
                />
              </div>
              <div className="md:col-span-2">
                <Select value={rarity} onValueChange={setRarity}>
                  <SelectTrigger><SelectValue placeholder="Rarity"/></SelectTrigger>
                  <SelectContent>
                    <SelectItem value="all">All rarities</SelectItem>
                    {(["Common","Uncommon","Rare","Ultra Rare","Secret Rare"] as const).map(r=> (
                      <SelectItem key={r} value={r}>{r}</SelectItem>
                    ))}
                  </SelectContent>
                </Select>
              </div>
              <div className="md:col-span-2">
                <Select value={sort} onValueChange={setSort}>
                  <SelectTrigger><SelectValue placeholder="Sort"/></SelectTrigger>
                  <SelectContent>
                    <SelectItem value="recommended">Recommended</SelectItem>
                    <SelectItem value="price-asc">Price: Low to High</SelectItem>
                    <SelectItem value="price-desc">Price: High to Low</SelectItem>
                    <SelectItem value="newest">Newest</SelectItem>
                  </SelectContent>
                </Select>
              </div>
              <div className="md:col-span-2 flex items-center gap-3 justify-between md:justify-end">
                <div className="flex items-center gap-2">
                  <Label htmlFor="graded" className="text-sm">Graded only</Label>
                  <Switch id="graded" checked={gradedOnly} onCheckedChange={setGradedOnly}/>
                </div>
                <Button variant="outline" className="gap-2"><Filter className="h-4 w-4"/>More</Button>
              </div>
            </div>

            <div className="mt-4">
              <Label className="text-xs text-neutral-500">Price range ({currency})</Label>
              <div className="flex items-center gap-3">
                <Slider value={priceRange} min={0} max={2000} step={10} onValueChange={setPriceRange}/>
                <div className="text-sm w-40 text-right">{formatMoney(priceRange[0], currency)} – {formatMoney(priceRange[1], currency)}</div>
              </div>
            </div>

            <div className="mt-3 flex flex-wrap gap-3">
              {(["NM","LP","MP","HP","DMG"] as const).map(c => (
                <div key={c} className="flex items-center gap-2">
                  <Checkbox id={`cond-${c}`} checked={conditions.includes(c)} onCheckedChange={(v)=>{
                    setConditions(s => v ? Array.from(new Set([...s,c])) : s.filter(x=>x!==c));
                  }}/>
                  <Label htmlFor={`cond-${c}`} className="text-sm">{c}</Label>
                </div>
              ))}
            </div>
          </CardContent>
        </Card>
      </section>

      {/* Results */}
      <main className="max-w-7xl mx-auto px-4 pb-20">
        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
          {filtered.map((item) => (
            <motion.div key={item.id} layout initial={{ opacity: 0, y: 10 }} animate={{ opacity: 1, y: 0 }}>
              <Card className="rounded-2xl shadow-sm hover:shadow-md transition">
                <CardHeader className="p-3">
                  <div className="flex items-center justify-between">
                    <CardTitle className="text-base font-semibold line-clamp-1">{item.name}</CardTitle>
                    <Badge className={rarityColors[item.rarity]}>{item.rarity}</Badge>
                  </div>
                  <CardDescription className="text-xs">{item.set} • #{item.number}</CardDescription>
                </CardHeader>
                <CardContent className="p-0">
                  <div className="aspect-[4/3] overflow-hidden rounded-xl m-3 bg-neutral-100">
                    <img src={item.image} alt={item.name} className="w-full h-full object-contain"/>
                  </div>
                </CardContent>
                <CardFooter className="p-3 flex items-center justify-between">
                  <div className="space-y-1">
                    <div className="font-bold text-lg">
                      {formatMoney(item.price / (RATES[item.currency] || 1) * (RATES[currency] || 1), currency)}
                    </div>
                    <div className="text-xs text-neutral-500 flex items-center gap-2">
                      <ShieldCheck className="h-3.5 w-3.5"/> {item.grading || "Raw"}{item.grade ? ` ${item.grade}` : ""}
                      <span>•</span>
                      <Globe2 className="h-3.5 w-3.5"/> {item.country}
                    </div>
                  </div>
                  <div className="flex gap-2">
                    <Button variant="outline" size="sm" className="gap-2" onClick={()=>{setDetail(item);}}>
                      <TrendingUp className="h-4 w-4"/> Price
                    </Button>
                    <Button size="sm" className="gap-2" onClick={()=>{setDetail(item); setOfferOpen(true);}}>
                      <Tag className="h-4 w-4"/> Offer
                    </Button>
                  </div>
                </CardFooter>
              </Card>
            </motion.div>
          ))}
        </div>
        {filtered.length === 0 && (
          <div className="text-center text-neutral-500 py-16">No cards match your filters yet.</div>
        )}
      </main>

      {/* New Listing Dialog */}
      <Dialog open={newListingOpen} onOpenChange={setNewListingOpen}>
        <DialogContent className="max-w-xl">
          <DialogHeader>
            <DialogTitle className="flex items-center gap-2"><Plus className="h-5 w-5"/> List a Card</DialogTitle>
            <DialogDescription>Quickly post a card for global buyers. Escrow & shipping labels are supported at checkout.</DialogDescription>
          </DialogHeader>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
            <div className="space-y-2">
              <Label>Name</Label>
              <Input placeholder="e.g., Charizard (Base Set Holo)"/>
            </div>
            <div className="space-y-2">
              <Label>Set</Label>
              <Input placeholder="e.g., Base Set (1999)"/>
            </div>
            <div className="space-y-2">
              <Label>Card No.</Label>
              <Input placeholder="e.g., 4/102"/>
            </div>
            <div className="space-y-2">
              <Label>Rarity</Label>
              <Select defaultValue="Rare">
                <SelectTrigger><SelectValue/></SelectTrigger>
                <SelectContent>
                  {(["Common","Uncommon","Rare","Ultra Rare","Secret Rare"] as const).map(r => (
                    <SelectItem key={r} value={r}>{r}</SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
            <div className="space-y-2">
              <Label>Condition</Label>
              <Select defaultValue="NM">
                <SelectTrigger><SelectValue/></SelectTrigger>
                <SelectContent>
                  {(["NM","LP","MP","HP","DMG"] as const).map(c => (
                    <SelectItem key={c} value={c}>{c}</SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
            <div className="space-y-2">
              <Label>Grading</Label>
              <Select defaultValue="Raw">
                <SelectTrigger><SelectValue/></SelectTrigger>
                <SelectContent>
                  {(["Raw","PSA","BGS","CGC"] as const).map(g => (
                    <SelectItem key={g} value={g}>{g}</SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
            <div className="space-y-2">
              <Label>Price</Label>
              <div className="flex gap-2">
                <Select defaultValue="USD">
                  <SelectTrigger className="w-28"><SelectValue/></SelectTrigger>
                  <SelectContent>
                    {(["USD","EUR","AED","GBP","JPY"] as const).map(c => (
                      <SelectItem key={c} value={c}>{c}</SelectItem>
                    ))}
                  </SelectContent>
                </Select>
                <Input placeholder="e.g., 299" type="number"/>
              </div>
            </div>
            <div className="space-y-2 md:col-span-2">
              <Label>Image URL</Label>
              <Input placeholder="https://..."/>
            </div>
            <div className="flex items-center justify-between md:col-span-2 p-3 rounded-xl bg-neutral-50 border">
              <div className="flex items-center gap-2 text-sm"><ShieldCheck className="h-4 w-4"/> Enable Escrow Protection</div>
              <Switch defaultChecked />
            </div>
          </div>
          <DialogFooter>
            <Button variant="outline">Cancel</Button>
            <Button className="gap-2"><SendHorizontal className="h-4 w-4"/> Post Listing</Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>

      {/* Detail Sheet */}
      <Sheet open={!!detail} onOpenChange={(o)=>!o && setDetail(null)}>
        <SheetContent side="right" className="w-full sm:max-w-xl">
          <SheetHeader>
            <SheetTitle className="flex items-center gap-2">
              <BadgePercent className="h-5 w-5"/> {detail?.name}
            </SheetTitle>
          </SheetHeader>
          {detail && (
            <div className="space-y-4 py-3">
              <div className="aspect-[4/3] rounded-xl overflow-hidden bg-neutral-100">
                <img src={detail.image} alt={detail.name} className="w-full h-full object-contain"/>
              </div>
              <div className="text-sm text-neutral-600">{detail.set} • #{detail.number}</div>
              <div className="flex items-center gap-2 text-sm">
                <Badge className={rarityColors[detail.rarity]}>{detail.rarity}</Badge>
                <Badge variant="secondary">{detail.condition}</Badge>
                <Badge variant="secondary">{detail.grading || "Raw"}{detail.grade ? ` ${detail.grade}` : ""}</Badge>
              </div>
              <Separator/>
              <div className="grid grid-cols-1 gap-3">
                <Card className="shadow-none border-neutral-200">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-sm">Recent Sales</CardTitle>
                    <CardDescription>Last 5 months (USD)</CardDescription>
                  </CardHeader>
                  <CardContent>
                    <div className="h-40">
                      <ResponsiveContainer width="100%" height="100%">
                        <LineChart data={detail.history} margin={{ left: 6, right: 6, top: 6 }}>
                          <XAxis dataKey="date" hide={true} />
                          <YAxis width={40} />
                          <Tooltip />
                          <Line type="monotone" dataKey="price" dot={false} strokeWidth={2} />
                        </LineChart>
                      </ResponsiveContainer>
                    </div>
                  </CardContent>
                </Card>
              </div>
              <div className="flex items-center justify-between">
                <div>
                  <div className="text-xs text-neutral-500">Listed Price</div>
                  <div className="text-2xl font-extrabold">
                    {formatMoney(detail.price / (RATES[detail.currency] || 1) * (RATES[currency] || 1), currency)}
                  </div>
                </div>
                <div className="flex items-center gap-3">
                  <div className="flex items-center gap-2 text-sm"><ShieldCheck className="h-4 w-4"/> Escrow</div>
                  <Button onClick={()=>setOfferOpen(true)} className="gap-2"><Tag className="h-4 w-4"/> Make Offer</Button>
                </div>
              </div>
            </div>
          )}
          <SheetFooter>
            <div className="text-xs text-neutral-500">Seller: {detail?.seller} • {detail?.country}</div>
          </SheetFooter>
        </SheetContent>
      </Sheet>

      {/* Offer Dialog */}
      <Dialog open={offerOpen} onOpenChange={setOfferOpen}>
        <DialogContent className="max-w-sm">
          <DialogHeader>
            <DialogTitle>Make an Offer</DialogTitle>
            <DialogDescription>Send a price to the seller. Funds are held in escrow until delivery is confirmed.</DialogDescription>
          </DialogHeader>
          <div className="space-y-3">
            <div className="space-y-2">
              <Label>Offer ({currency})</Label>
              <Input type="number" placeholder="e.g., 650"/>
            </div>
            <div className="flex items-center justify-between p-3 rounded-xl bg-neutral-50 border">
              <div className="flex items-center gap-2 text-sm"><ShieldCheck className="h-4 w-4"/> Escrow Protection</div>
              <Switch checked={escrow} onCheckedChange={setEscrow}/>
            </div>
          </div>
          <DialogFooter>
            <Button variant="outline" onClick={()=>setOfferOpen(false)}>Cancel</Button>
            <Button className="gap-2"><Coins className="h-4 w-4"/> Send Offer</Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>

      {/* Footer */}
      <footer className="border-t bg-white/60 backdrop-blur">
        <div className="max-w-7xl mx-auto px-4 py-6 grid grid-cols-1 md:grid-cols-3 gap-4 items-center">
          <div className="text-sm text-neutral-600">© {new Date().getFullYear()} PokéSwap — Buy & Sell Pokémon cards worldwide.</div>
          <div className="text-sm text-neutral-500">Managed payments, currency conversion, built-in shipping labels, fraud checks.</div>
          <div className="md:text-right text-sm">
            <Tabs defaultValue="buyer">
              <TabsList>
                <TabsTrigger value="buyer">Buyer</TabsTrigger>
                <TabsTrigger value="seller">Seller</TabsTrigger>
              </TabsList>
              <TabsContent value="buyer" className="text-neutral-600">Protected by escrow. Authenticity optional via grading partners.</TabsContent>
              <TabsContent value="seller" className="text-neutral-600">Instant labels. Low fees. Auto-relist & counteroffers.</TabsContent>
            </Tabs>
          </div>
        </div>
      </footer>
    </div>
  );
}

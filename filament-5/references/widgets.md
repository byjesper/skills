# Filament 5 - Widgets

> **Namespace**: `Filament\Widgets`

---


## Stats Overview

```php
use Filament\Widgets\StatsOverviewWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;

class RevenueStats extends StatsOverviewWidget
{
    protected function getStats(): array
    {
        return [
            Stat::make('Revenue', '$192k')
                ->description('32% increase')
                ->descriptionIcon('heroicon-m-arrow-trending-up')
                ->color('success')
                ->chart([7, 3, 4, 5, 6, 3, 5, 3]),
        ];
    }
}
```

## Chart Widgets

```php
use Filament\Widgets\ChartWidget;

class BlogPostsChart extends ChartWidget
{
    protected static ?string $heading = 'Blog Posts';
    protected static string $color = 'primary';

    protected function getData(): array
    {
        // PREFER database queries over hardcoded arrays
        $postsPerMonth = BlogPost::selectRaw('MONTH(created_at) as month, COUNT(*) as count')
            ->whereYear('created_at', now()->year)
            ->groupBy('month')
            ->orderBy('month')
            ->pluck('count', 'month');

        return [
            'datasets' => [
                [
                    'label' => 'Posts',
                    'data' => collect(range(1, 12))
                        ->map(fn (int $month) => $postsPerMonth[$month] ?? 0)
                        ->all(),
                ],
            ],
            'labels' => ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun',
                         'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'],
        ];
    }

    protected function getType(): string { return 'line'; } // line, bar, pie, doughnut, scatter, bubble, radar, polarArea
}
```

> **Always use database queries for widget data.** Hardcoded arrays are only acceptable for demos or when no database model exists. Use `Trend` (from `flowframe/laravel-trend` package) for time-series aggregation, or write Eloquent queries for counts, averages, and grouped data.

Widget types: Custom (Livewire+Blade), StatsOverview, ChartWidget, TableWidget.


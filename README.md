# Blazor client notifications on database record change
This example uses .NET CORE 3.0 Blazor server side to real-time update a HTML page on any database record changes.

<img src="https://github.com/christiandelbianco/blazor-notification-db-record-change/blob/master/Schema.png" />

Based on the observer design pattern in which an object, called the subject, maintains a list of its dependents, called observers, and notifies them automatically of any state changes, usually by calling one of their methods. 

<img src="https://github.com/christiandelbianco/blazor-notification-db-record-change/blob/master/2019-11-03at21-05-44.gif" />

## Database table
Table for which I want to receive notifications every time its content changes:

```SQL
CREATE TABLE [dbo].[WeatherForecasts](
	[City] [nvarchar](50) NOT NULL,
	[Temperature] [int] NOT NULL
)
```

## Subject
Singleton instance wrapping SqlTableDependency and forwarding record table changes to subscribers:

```C#
    public delegate void WeatherForecastDelegate(object sender, WeatherForecastChangeEventArgs args);

    public class WeatherForecastChangeEventArgs : EventArgs
    {
        public WeatherForecast NewWeatherForecast { get; }
        public WeatherForecast OldWeatherForecast { get; }

        public WeatherForecastChangeEventArgs(WeatherForecast newWeatherForecast, WeatherForecast oldWeatherForecast)
        {
            this.NewWeatherForecast = newWeatherForecast;
            this.OldWeatherForecast = oldWeatherForecast;
        }
    }

    public interface IWeatherForecastService
    {
        public event WeatherForecastDelegate OnWeatherForecastChanged;
        IList<WeatherForecast> GetForecast();
    }

    public class WeatherForecastService : IWeatherForecastService, IDisposable
    {
        private const string TableName = "WeatherForecasts";
        private SqlTableDependency<WeatherForecast> _notifier;
        private IConfiguration _configuration;
        
        public event WeatherForecastDelegate OnWeatherForecastChanged;

        public WeatherForecastService(IConfiguration configuration)
        {
            _configuration = configuration;

            _notifier = new SqlTableDependency<WeatherForecast>(_configuration["ConnectionString"], TableName);
            _notifier.OnChanged += this.TableDependency_Changed;
            _notifier.Start();
        }

        private void TableDependency_Changed(object sender, RecordChangedEventArgs<WeatherForecast> e)
        { 
            if (this.OnWeatherForecastChanged != null)
            {
                this.OnWeatherForecastChanged(this, new WeatherForecastChangeEventArgs(e.Entity, e.EntityOldValues));
            }
        }

        public IList<WeatherForecast> GetForecast()
        {
            var result = new List<WeatherForecast>();

            using (var sqlConnection = new SqlConnection(_configuration["ConnectionString"]))
            {
                sqlConnection.Open();

                using (var command = sqlConnection.CreateCommand())
                {
                    command.CommandText = "SELECT * FROM " + TableName;
                    command.CommandType = CommandType.Text;

                    using (SqlDataReader reader = command.ExecuteReader())
                    {
                        if (reader.HasRows)
                        {
                            while (reader.Read())
                            {
                                result.Add(new WeatherForecast
                                {
                                    City = reader.GetString(reader.GetOrdinal("City")),
                                    Temperature = reader.GetInt32(reader.GetOrdinal("Temperature"))
                                });
                            }
                        }
                    }
                }
            }

            return result;
        }

        public void Dispose()
        {
            _notifier.Stop();
            _notifier.Dispose();
        }
    }
```

Registration as Singleton:

```C#
public void ConfigureServices(IServiceCollection services)
{
    ...
    ...
    services.AddSingleton<IWeatherForecastService, WeatherForecastService>();
}
```

## Observer
Index.razor page code (event subscriber):

```C#
@page "/"

@using DataBaseRecordChaneNotificationWithBlazor.Data

@inject IWeatherForecastService ForecastService
@implements IDisposable

<h1>Weather forecast</h1>

<p>Immediate client notification on record table change with Blazor</p>

<table class="table">
    <thead>
        <tr>
            <th>City</th>
            <th>Temp. (C)</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var forecast in forecasts)
        {
            <tr>
                <td>@forecast.City</td>
                <td>@forecast.Temperature</td>
            </tr>
        }
    </tbody>
</table>

@code {
    IList<WeatherForecast> forecasts;

    protected override void OnInitialized()
    {
        this.ForecastService.OnWeatherForecastChanged += this.WeatherForecastChanged;
        this.forecasts = this.ForecastService.GetForecast();
    }

    private async void WeatherForecastChanged(object sender, WeatherForecastChangeEventArgs args)
    {
        var recordToupdate = this.forecasts.FirstOrDefault(x => x.City == args.NewWeatherForecast.City);
        if (recordToupdate == null)
        {
            this.forecasts.Add(args.NewWeatherForecast);
        }
        else
        {
            recordToupdate.Temperature = args.NewWeatherForecast.Temperature;
        }

        await InvokeAsync(() =>
        {
            base.StateHasChanged();
        });
    }

    public void Dispose()
    {
        this.ForecastService.OnWeatherForecastChanged += this.WeatherForecastChanged;
    }
}
```

More info on https://github.com/christiandelbianco/monitor-table-change-with-sqltabledependency.

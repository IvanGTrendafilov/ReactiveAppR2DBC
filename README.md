# ReactiveAppR2DBC

import com.fasterxml.jackson.databind.ObjectMapper;
import com.theopentag.sportsbook.betprocessor.components.BetResponseToJsonConverter;
import com.theopentag.sportsbook.betprocessor.components.JsonToBetResponseConverter;
import com.theopentag.sportsbook.betprocessor.repositories.BalanceDataRequestRepository;
import com.theopentag.sportsbook.betprocessor.repositories.BetRepository;
import com.theopentag.sportsbook.betprocessor.repositories.BetRequestRepository;
import com.theopentag.sportsbook.betprocessor.repositories.BetResponseRepository;
import com.theopentag.sportsbook.betprocessor.repositories.LineRepository;
import io.r2dbc.postgresql.PostgresqlConnectionConfiguration;
import io.r2dbc.postgresql.PostgresqlConnectionFactory;
import io.r2dbc.spi.ConnectionFactory;
import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.data.r2dbc.config.AbstractR2dbcConfiguration;
import org.springframework.data.r2dbc.connectionfactory.R2dbcTransactionManager;
import org.springframework.data.r2dbc.convert.R2dbcCustomConversions;
import org.springframework.data.r2dbc.core.DatabaseClient;
import org.springframework.data.r2dbc.core.DefaultReactiveDataAccessStrategy;
import org.springframework.data.r2dbc.dialect.PostgresDialect;
import org.springframework.data.r2dbc.repository.config.EnableR2dbcRepositories;
import org.springframework.data.r2dbc.repository.support.R2dbcRepositoryFactory;
import org.springframework.transaction.ReactiveTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import org.springframework.transaction.reactive.TransactionalOperator;

import java.util.ArrayList;
import java.util.List;

@Configuration
@EnableR2dbcRepositories
@EnableTransactionManagement
public class R2dbcPostgreSQLConfig extends AbstractR2dbcConfiguration {

    private final R2dbcPostgreSQLConfig.PostgreSqlReactiveConfig postgreSqlReactiveConfig;
    private final ObjectMapper objectMapper;

    public R2dbcPostgreSQLConfig(final PostgreSqlReactiveConfig postgreSqlReactiveConfig,
                                 final ObjectMapper objectMapper) {
        this.postgreSqlReactiveConfig = postgreSqlReactiveConfig;
        this.objectMapper = objectMapper;
    }

    @ConfigurationProperties(prefix = "datasource")
    @Getter
    @Setter
    public static class PostgreSqlReactiveConfig {
        private String host;
        private String database;
        private String username;
        private int port;
        private String password;
    }

    @Override
    @Bean
    public PostgresqlConnectionFactory connectionFactory() {
        return new PostgresqlConnectionFactory(PostgresqlConnectionConfiguration
                .builder()
                .host(postgreSqlReactiveConfig.getHost())
                .database(postgreSqlReactiveConfig.getDatabase())
                .username(postgreSqlReactiveConfig.getUsername())
                .password(postgreSqlReactiveConfig.getPassword())
                .port(postgreSqlReactiveConfig.getPort())
                .build());
    }

    @Bean
    public DatabaseClient databaseClient() {
        return DatabaseClient.create(connectionFactory());
    }

    @Bean
    public BetResponseRepository betResponseRepository() {
        return r2dbcRepositoryFactory().getRepository(BetResponseRepository.class);
    }

    @Bean
    public BetRequestRepository betRequestRepository() {
        return r2dbcRepositoryFactory().getRepository(BetRequestRepository.class);
    }

    @Bean
    public BetRepository betRepository() {
        return r2dbcRepositoryFactory().getRepository(BetRepository.class);
    }

    @Bean
    public LineRepository lineRepository() {
        return r2dbcRepositoryFactory().getRepository(LineRepository.class);
    }

    @Bean
    public BalanceDataRequestRepository balanceDateRequestRepository() {
        return r2dbcRepositoryFactory().getRepository(BalanceDataRequestRepository.class);
    }

    @Bean
    public R2dbcRepositoryFactory r2dbcRepositoryFactory() {
        return new R2dbcRepositoryFactory(databaseClient(),
                new DefaultReactiveDataAccessStrategy(new PostgresDialect()));
    }

    @Bean
    @Override
    public R2dbcCustomConversions r2dbcCustomConversions() {
        final List<Converter<?, ?>> converters = new ArrayList<>();
        converters.add(new BetResponseToJsonConverter(objectMapper));
        converters.add(new JsonToBetResponseConverter(objectMapper));
        return new R2dbcCustomConversions(getStoreConversions(), converters);
    }

    @Bean
    public ReactiveTransactionManager reactiveTransactionManager(final ConnectionFactory connectionFactory) {
        return new R2dbcTransactionManager(connectionFactory);
    }

    @Bean
    public TransactionalOperator transactionalOperator(final ReactiveTransactionManager reactiveTransactionManager) {
        return TransactionalOperator.create(reactiveTransactionManager);
    }
}

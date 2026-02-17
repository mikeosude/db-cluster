# Database Cluster Configurations

This repository contains database configuration files and setup scripts for Percona XtraDB Cluster and PostgreSQL deployments.

## Overview

A centralized location for managing and versioning database configurations for:
- **Percona XtraDB Cluster** - High-availability MySQL clustering solution
- **PostgreSQL** - Advanced open-source relational database

## Repository Structure

```
db-cluster/
├── percona-xtradb/
│   ├── production/
│   ├── staging/
│   └── development/
├── postgresql/
│   ├── production/
│   ├── staging/
│   └── development/
├── scripts/          # Setup and maintenance scripts
└── docs/            # Additional documentation
```

## Percona XtraDB Cluster

Percona XtraDB Cluster provides synchronous multi-master replication for MySQL/MariaDB.

### Key Configuration Files
- `my.cnf` - Main configuration file
- `wsrep.cnf` - Galera cluster configuration
- Replication and cluster-specific settings

### Important Parameters
- `wsrep_cluster_address` - Cluster node addresses
- `wsrep_node_address` - Individual node address
- `wsrep_sst_method` - State snapshot transfer method

## PostgreSQL

PostgreSQL configurations for standalone and clustered deployments.

### Key Configuration Files
- `postgresql.conf` - Main server configuration
- `pg_hba.conf` - Client authentication
- `recovery.conf` / `standby.signal` - Replication settings

### Common Settings
- Connection pooling parameters
- Replication configuration
- Performance tuning settings
- Backup and recovery options

## Getting Started

### Prerequisites

**For Percona XtraDB Cluster:**
- Percona XtraDB Cluster 8.0+ installed
- Network connectivity between cluster nodes
- Sufficient disk space for SST operations

**For PostgreSQL:**
- PostgreSQL 12+ installed
- `pg_basebackup` for replication setup
- Appropriate client tools (psql, pgAdmin, etc.)

### Usage

1. Clone this repository:
   ```bash
   git clone https://github.com/mikeosude/db-cluster.git
   cd db-cluster
   ```

2. Navigate to the appropriate database and environment directory

3. Copy configuration files to your database installation:
   ```bash
   # Percona XtraDB
   sudo cp percona-xtradb/production/my.cnf /etc/mysql/
   
   # PostgreSQL
   sudo cp postgresql/production/postgresql.conf /etc/postgresql/14/main/
   ```

4. Update environment-specific values

5. Restart database services to apply changes

## Security Notes

⚠️ **Critical Security Practices:**
- **Never commit passwords, API keys, or sensitive credentials**
- Use environment variables for sensitive configuration
- Implement secret management (HashiCorp Vault, AWS Secrets Manager, etc.)
- Ensure `.gitignore` excludes sensitive files
- Use SSL/TLS for database connections
- Restrict network access with firewalls

## Environment Variables

Create a `.env` file (not tracked in git) for sensitive values:

```bash
# Percona XtraDB Cluster
MYSQL_ROOT_PASSWORD=your_root_password
WSREP_SST_PASSWORD=your_sst_password
MYSQL_REPLICATION_PASSWORD=your_replication_password

# PostgreSQL
POSTGRES_PASSWORD=your_postgres_password
POSTGRES_REPLICATION_PASSWORD=your_replication_password
PG_CONNECTION_STRING=postgresql://user:pass@host:5432/dbname
```

## Configuration Management

### Version Control Best Practices
- Use separate branches for different environments
- Tag stable configurations for rollback capability
- Document all parameter changes in commit messages
- Review changes before applying to production

### Testing Configurations
1. Always test in development environment first
2. Validate syntax before applying
3. Monitor performance after changes
4. Keep rollback configurations ready

## Cluster Management

### Percona XtraDB Cluster Operations
```bash
# Check cluster status
mysql -e "SHOW STATUS LIKE 'wsrep_%';"

# Bootstrap cluster
systemctl start mysql@bootstrap.service

# Join node to cluster
systemctl start mysql
```

### PostgreSQL Replication
```bash
# Check replication status
psql -c "SELECT * FROM pg_stat_replication;"

# Promote standby to primary
pg_ctl promote -D /var/lib/postgresql/data
```

## Backup Strategy

- Regular automated backups scheduled
- Off-site backup storage configured
- Backup restoration tested regularly
- Configuration files versioned in this repository

## Monitoring

Recommended monitoring metrics:
- Cluster health and node status
- Replication lag
- Connection pool utilization
- Query performance
- Disk space and I/O metrics

## Contributing

1. Create a feature branch: `git checkout -b config/description`
2. Make and test your changes
3. Commit with descriptive messages: `git commit -m "Update PostgreSQL max_connections for prod"`
4. Push and create a pull request

## Maintenance Schedule

- Weekly: Review logs and performance metrics
- Monthly: Update documentation and audit configurations
- Quarterly: Review and update security settings
- Yearly: Database version upgrades and major configuration reviews

## Troubleshooting

### Percona XtraDB Cluster
- Check `/var/log/mysql/error.log` for cluster issues
- Verify network connectivity between nodes
- Ensure SST ports are open (4444, 4567, 4568)

### PostgreSQL
- Check logs in `/var/log/postgresql/`
- Verify replication slots: `SELECT * FROM pg_replication_slots;`
- Monitor WAL generation and archiving

## Resources

- [Percona XtraDB Cluster Documentation](https://www.percona.com/doc/percona-xtradb-cluster/)
- [PostgreSQL Official Documentation](https://www.postgresql.org/docs/)
- [Galera Cluster Documentation](https://galeracluster.com/library/documentation/)

## License

[Specify your license here]

## Contact

For questions or issues, please open an issue in this repository.

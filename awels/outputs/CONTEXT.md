# Shopizer Repository Overview

## Main Purpose
This Shopizer repository (version 1.1.5) is a legacy Java-based e-commerce platform providing online sales management capabilities. It includes an online catalog, shopping cart, order fulfillment, and online invoicing functionalities. This is an older version of Shopizer built with traditional Java EE technologies.

## Key Features and Functionality
- **Online Catalog Management**: Product catalog with categories, descriptions, and media management
- **Shopping Cart**: Complete shopping cart functionality for customer purchases
- **Order Fulfillment**: Order processing and management capabilities
- **Online Invoicing**: Invoice generation and management
- **Administration Interface**: Separate administration application (sm-central) for store management
- **Customer-Facing Storefront**: Public-facing shop application (sm-shop) for browsing and purchasing
- **Media Management**: Dedicated media application for hosting product images and downloadable files
- **Multi-Database Support**: Configurable support for MySQL, Oracle, and HSQLDB databases

## Technologies Used
- **Java**: Core language, requires JDK 1.5 or higher (Java 5+)
- **Apache Ant**: Build system for compiling and packaging applications
- **Hibernate 3**: ORM framework for database persistence with Hibernate Search integration
- **Spring Framework**: Dependency injection and application framework (classic Spring, not Spring Boot)
- **Struts 2**: MVC web framework (version 2.2.1.1) with jQuery plugin support
- **Apache Axis & JAX-WS**: Web services support
- **Lucene 2.3.0**: Full-text search capabilities via Hibernate Search
- **C3P0**: Database connection pooling
- **Freemarker**: Template engine for dynamic content generation
- **OSCache**: Hibernate second-level cache provider

## Architecture Overview
The application follows a traditional three-tier Java EE architecture with three main modules:

1. **sm-core**: Core business logic library packaged as a JAR file
   - Contains entity models, service layer, and data access objects
   - Hibernate mappings for ORM
   - Shared configuration and resources
   - Built using Apache Ant

2. **sm-central**: Administration web application (WAR)
   - Store management interface
   - Depends on sm-core
   - Built with Struts 2 framework
   - Accessible at `/central` context path

3. **sm-shop**: Customer-facing storefront web application (WAR)
   - Public catalog and shopping interface
   - Depends on sm-core
   - Built with Struts 2 framework
   - Accessible at `/shop` context path

4. **media**: Static media server application (WAR)
   - Hosts product images, branding, and downloadable files
   - Separate deployment for media content

## Database Support
- **MySQL**: Primary supported database with full schema
- **Oracle**: Supported with dedicated schema (Oracle XE/APEX)
- **HSQLDB**: In-memory database for testing and development

## Build and Deployment
- Built using Apache Ant with separate build.xml files for each module
- Generates WAR files for deployment to Java application servers (e.g., Apache Tomcat)
- Configuration through properties files in `sm-core/conf/properties/`
- Recommended JVM settings: `-Xms256m -Xmx256m -XX:MaxPermSize=128m`

## Additional Information
- **Version**: 1.1.5 (legacy version)
- **License**: GNU Lesser General Public License (LGPL) Version 3
- **Upgrade Path**: Schema upgrade scripts available for versions 1.1.2 through 1.1.5
- **Configuration**: Requires manual configuration of database connections, SMTP settings, and media paths before deployment

**Note**: This is a legacy version of Shopizer. Modern versions of Shopizer (3.x+) use Spring Boot, REST APIs, microservices architecture, and different technology stacks. This repository represents the older monolithic architecture.
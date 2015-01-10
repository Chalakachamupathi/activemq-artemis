# Paging

ActiveMQ transparently supports huge queues containing millions of
messages while the server is running with limited memory.

In such a situation it's not possible to store all of the queues in
memory at any one time, so ActiveMQ transparently *pages* messages into
and out of memory as they are needed, thus allowing massive queues with
a low memory footprint.

ActiveMQ will start paging messages to disk, when the size of all
messages in memory for an address exceeds a configured maximum size.

By default, ActiveMQ does not page messages - this must be explicitly
configured to activate it.

## Page Files

Messages are stored per address on the file system. Each address has an
individual folder where messages are stored in multiple files (page
files). Each file will contain messages up to a max configured size
(`page-size-bytes`). The system will navigate on the files as needed,
and it will remove the page file as soon as all the messages are
acknowledged up to that point.

Browsers will read through the page-cursor system.

Consumers with selectors will also navigate through the page-files and
it will ignore messages that don't match the criteria.

## Configuration

You can configure the location of the paging folder

Global paging parameters are specified on the main configuration file
(`activemq-configuration.xml`).

    <configuration xmlns="urn:activemq"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="urn:activemq /schema/activemq-configuration.xsd">
    ...
    <paging-directory>/somewhere/paging-directory</paging-directory>
    ...

  Property Name        Description                                                                                                                 Default
  -------------------- --------------------------------------------------------------------------------------------------------------------------- -------------
  `paging-directory`   Where page files are stored. ActiveMQ will create one folder for each address being paged under this configured location.   data/paging

  : Paging Configuration Parameters

## Paging Mode

As soon as messages delivered to an address exceed the configured size,
that address alone goes into page mode.

> **Note**
>
> Paging is done individually per address. If you configure a
> max-size-bytes for an address, that means each matching address will
> have a maximum size that you specified. It DOES NOT mean that the
> total overall size of all matching addresses is limited to
> max-size-bytes.

## Configuration

Configuration is done at the address settings, done at the main
configuration file (`activemq-configuration.xml`).

    <address-settings>
       <address-setting match="jms.someaddress">
          <max-size-bytes>104857600</max-size-bytes>
          <page-size-bytes>10485760</page-size-bytes>
          <address-full-policy>PAGE</address-full-policy>
       </address-setting>
    </address-settings>

This is the list of available parameters on the address settings.

  Property Name           Description                                                                                                                                                                                                                                                                                                                                                                                                        Default
  ----------------------- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ ----------------------------------
  `max-size-bytes`        What's the max memory the address could have before entering on page mode.                                                                                                                                                                                                                                                                                                                                         -1 (disabled)
  `page-size-bytes`       The size of each page file used on the paging system                                                                                                                                                                                                                                                                                                                                                               10MiB (10 \* 1024 \* 1024 bytes)
  `address-full-policy`   This must be set to PAGE for paging to enable. If the value is PAGE then further messages will be paged to disk. If the value is DROP then further messages will be silently dropped. If the value is FAIL then the messages will be dropped and the client message producers will receive an exception. If the value is BLOCK then client message producers will block when they try and send further messages.   PAGE
  `page-max-cache-size`   The system will keep up to \<`page-max-cache-size` page files in memory to optimize IO during paging navigation.                                                                                                                                                                                                                                                                                                   5

  : Paging Address Settings

## Dropping messages

Instead of paging messages when the max size is reached, an address can
also be configured to just drop messages when the address is full.

To do this just set the `address-full-policy` to `DROP` in the address
settings

## Dropping messages and throwing an exception to producers

Instead of paging messages when the max size is reached, an address can
also be configured to drop messages and also throw an exception on the
client-side when the address is full.

To do this just set the `address-full-policy` to `FAIL` in the address
settings

## Blocking producers

Instead of paging messages when the max size is reached, an address can
also be configured to block producers from sending further messages when
the address is full, thus preventing the memory being exhausted on the
server.

When memory is freed up on the server, producers will automatically
unblock and be able to continue sending.

To do this just set the `address-full-policy` to `BLOCK` in the address
settings

In the default configuration, all addresses are configured to block
producers after 10 MiB of data are in the address.

## Caution with Addresses with Multiple Queues

When a message is routed to an address that has multiple queues bound to
it, e.g. a JMS subscription in a Topic, there is only 1 copy of the
message in memory. Each queue only deals with a reference to this.
Because of this the memory is only freed up once all queues referencing
the message have delivered it.

If you have a single lazy subscription, the entire address will suffer
IO performance hit as all the queues will have messages being sent
through an extra storage on the paging system.

For example:

-   An address has 10 queues

-   One of the queues does not deliver its messages (maybe because of a
    slow consumer).

-   Messages continually arrive at the address and paging is started.

-   The other 9 queues are empty even though messages have been sent.

In this example all the other 9 queues will be consuming messages from
the page system. This may cause performance issues if this is an
undesirable state.

## Example

See ? for an example which shows how to use paging with ActiveMQ.